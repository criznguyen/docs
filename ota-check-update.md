# C Language Firmware Update Guide for IoT Devices

## Overview

This guide provides step-by-step instructions for implementing firmware update functionality in C for IoT devices (ESP32, STM32, ESP8266, and other embedded systems). The implementation includes checking for updates and downloading firmware using the OTAP service.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Setup HTTP Client](#step-1-setup-http-client)
3. [Step 2: Check for Firmware Updates](#step-2-check-for-firmware-updates)
4. [Step 3: Download Firmware](#step-3-download-firmware)
5. [Step 4: Verify Firmware](#step-4-verify-firmware)
6. [Step 5: Complete Workflow](#step-5-complete-workflow)
7. [Error Handling](#error-handling)
8. [Best Practices](#best-practices)

## Prerequisites

### Required Libraries

**For ESP32:**
- `esp_http_client.h` (ESP-IDF)
- `esp_https_ota.h` (ESP-IDF)
- `mbedtls/sha256.h` (for SHA256 verification)
- `cJSON.h` (for JSON parsing)

**For STM32:**
- Custom HTTP client library (e.g., LwIP with HTTP client)
- `mbedtls` or `wolfSSL` for TLS
- JSON parser (e.g., `cJSON`)

**For ESP8266:**
- Arduino HTTPClient library
- Arduino JSON library
- SHA256 library

### Configuration

Define the following constants in your code:

```c
// API Configuration
#define API_BASE_URL "https://ota-link.theio.vn/api/v1"
#define DEVICE_ID "your-device-id"
#define PRODUCT_NAME "your-product-name"
#define CURRENT_VERSION "1.0.0"

// Network Configuration
#define WIFI_SSID "your-wifi-ssid"
#define WIFI_PASSWORD "your-wifi-password"

// Update Configuration
#define FIRMWARE_BUFFER_SIZE 4096
#define MAX_FIRMWARE_SIZE (1024 * 1024) // 1MB
```

## Step 1: Setup HTTP Client

### ESP32 Example

```c
#include "esp_http_client.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "wifi_connect.h"

static const char *TAG = "OTA_UPDATE";

// HTTP client configuration
esp_http_client_config_t config = {
    .url = API_BASE_URL,
    .timeout_ms = 30000,
    .skip_cert_common_name_check = false,
    .cert_pem = NULL, // Add certificate if using HTTPS
};

esp_http_client_handle_t http_client_init(const char *url) {
    esp_http_client_config_t config = {
        .url = url,
        .timeout_ms = 30000,
        .skip_cert_common_name_check = false,
    };
    
    return esp_http_client_init(&config);
}
```

### STM32 Example (Using LwIP)

```c
#include "lwip/err.h"
#include "lwip/sockets.h"
#include "lwip/dns.h"
#include "lwip/netdb.h"

#define HTTP_PORT 443

int http_connect(const char *host, uint16_t port) {
    int sock;
    struct sockaddr_in server_addr;
    struct hostent *server;
    
    // Resolve hostname
    server = gethostbyname(host);
    if (server == NULL) {
        return -1;
    }
    
    // Create socket
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        return -1;
    }
    
    // Configure server address
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    memcpy(&server_addr.sin_addr.s_addr, server->h_addr, server->h_length);
    
    // Connect
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        close(sock);
        return -1;
    }
    
    return sock;
}
```

## Step 2: Check for Firmware Updates

This step sends a POST request to check if a firmware update is available.

### Function Implementation

```c
#include <string.h>
#include <stdlib.h>
#include "cJSON.h"

typedef struct {
    bool update_available;
    char firmware_id[64];
    char version[32];
    int64_t size;
    char sha256[65];
    char download_url[256];
    int mtu;
    int anti_rollback;
    int min_battery;
    int min_signal;
    char campaign_id[64];
} firmware_update_info_t;

/**
 * Check for firmware updates
 * Returns 0 on success, negative on error
 */
int check_for_update(const char *device_id, const char *current_version, 
                     firmware_update_info_t *update_info) {
    char url[256];
    char request_body[512];
    char response_buffer[2048];
    int response_len = 0;
    
    // Construct URL
    snprintf(url, sizeof(url), "%s/devices/%s/check", API_BASE_URL, device_id);
    
    // Construct request body
    snprintf(request_body, sizeof(request_body),
        "{"
        "\"current_version\":\"%s\","
        "\"capabilities\":{"
        "\"battery_level\":%d,"
        "\"signal_strength\":%d,"
        "\"storage_free\":%lu"
        "}"
        "}",
        current_version,
        get_battery_level(),      // Your function to get battery
        get_signal_strength(),    // Your function to get signal
        get_free_storage()        // Your function to get free storage
    );
    
    ESP_LOGI(TAG, "Checking for updates: %s", url);
    
    // Initialize HTTP client
    esp_http_client_handle_t client = http_client_init(url);
    if (client == NULL) {
        ESP_LOGE(TAG, "Failed to initialize HTTP client");
        return -1;
    }
    
    // Set request method and headers
    esp_http_client_set_method(client, HTTP_METHOD_POST);
    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_post_field(client, request_body, strlen(request_body));
    
    // Perform request
    esp_err_t err = esp_http_client_perform(client);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "HTTP POST request failed: %s", esp_err_to_name(err));
        esp_http_client_cleanup(client);
        return -1;
    }
    
    // Read response
    int content_length = esp_http_client_get_content_length(client);
    if (content_length > 0 && content_length < sizeof(response_buffer)) {
        response_len = esp_http_client_read_response(client, response_buffer, 
                                                      sizeof(response_buffer) - 1);
        response_buffer[response_len] = '\0';
    }
    
    int status_code = esp_http_client_get_status_code(client);
    esp_http_client_cleanup(client);
    
    if (status_code != 200) {
        ESP_LOGE(TAG, "HTTP request failed with status: %d", status_code);
        return -1;
    }
    
    // Parse JSON response
    return parse_update_response(response_buffer, update_info);
}

/**
 * Parse update check response
 */
int parse_update_response(const char *json_str, firmware_update_info_t *info) {
    cJSON *root = cJSON_Parse(json_str);
    if (root == NULL) {
        ESP_LOGE(TAG, "Failed to parse JSON");
        return -1;
    }
    
    // Check if update is available
    cJSON *update_available = cJSON_GetObjectItem(root, "update_available");
    if (update_available == NULL || !cJSON_IsTrue(update_available)) {
        info->update_available = false;
        cJSON_Delete(root);
        return 0;
    }
    
    info->update_available = true;
    
    // Parse firmware information
    cJSON *firmware_id = cJSON_GetObjectItem(root, "firmware_id");
    cJSON *version = cJSON_GetObjectItem(root, "version");
    cJSON *size = cJSON_GetObjectItem(root, "size");
    cJSON *sha256 = cJSON_GetObjectItem(root, "sha256");
    cJSON *download_url = cJSON_GetObjectItem(root, "download_url");
    cJSON *mtu = cJSON_GetObjectItem(root, "mtu");
    cJSON *anti_rollback = cJSON_GetObjectItem(root, "anti_rollback");
    cJSON *min_battery = cJSON_GetObjectItem(root, "min_battery");
    cJSON *min_signal = cJSON_GetObjectItem(root, "min_signal");
    cJSON *campaign_id = cJSON_GetObjectItem(root, "campaign_id");
    
    if (firmware_id && cJSON_IsString(firmware_id)) {
        strncpy(info->firmware_id, firmware_id->valuestring, sizeof(info->firmware_id) - 1);
    }
    if (version && cJSON_IsString(version)) {
        strncpy(info->version, version->valuestring, sizeof(info->version) - 1);
    }
    if (size && cJSON_IsNumber(size)) {
        info->size = size->valueint;
    }
    if (sha256 && cJSON_IsString(sha256)) {
        strncpy(info->sha256, sha256->valuestring, sizeof(info->sha256) - 1);
    }
    if (download_url && cJSON_IsString(download_url)) {
        strncpy(info->download_url, download_url->valuestring, sizeof(info->download_url) - 1);
    }
    if (mtu && cJSON_IsNumber(mtu)) {
        info->mtu = mtu->valueint;
    }
    if (anti_rollback && cJSON_IsNumber(anti_rollback)) {
        info->anti_rollback = anti_rollback->valueint;
    }
    if (min_battery && cJSON_IsNumber(min_battery)) {
        info->min_battery = min_battery->valueint;
    }
    if (min_signal && cJSON_IsNumber(min_signal)) {
        info->min_signal = min_signal->valueint;
    }
    if (campaign_id && cJSON_IsString(campaign_id)) {
        strncpy(info->campaign_id, campaign_id->valuestring, sizeof(info->campaign_id) - 1);
    }
    
    cJSON_Delete(root);
    return 0;
}
```

### Usage Example

```c
void check_update_task(void *pvParameters) {
    firmware_update_info_t update_info = {0};
    
    ESP_LOGI(TAG, "Checking for firmware updates...");
    
    int ret = check_for_update(DEVICE_ID, CURRENT_VERSION, &update_info);
    if (ret != 0) {
        ESP_LOGE(TAG, "Failed to check for updates");
        return;
    }
    
    if (update_info.update_available) {
        ESP_LOGI(TAG, "Update available: %s (v%s)", 
                 update_info.firmware_id, update_info.version);
        ESP_LOGI(TAG, "Size: %lld bytes", update_info.size);
        ESP_LOGI(TAG, "Download URL: %s", update_info.download_url);
        
        // Proceed to download
        download_firmware(&update_info);
    } else {
        ESP_LOGI(TAG, "No update available");
    }
}
```

## Step 3: Download Firmware

Download firmware with support for resumable downloads using HTTP Range requests.

### Basic Download Function

```c
#include "mbedtls/sha256.h"

/**
 * Download firmware file
 * Returns 0 on success, negative on error
 */
int download_firmware(const firmware_update_info_t *update_info, 
                     const char *save_path) {
    ESP_LOGI(TAG, "Starting firmware download from: %s", update_info->download_url);
    
    // Initialize HTTP client
    esp_http_client_handle_t client = http_client_init(update_info->download_url);
    if (client == NULL) {
        ESP_LOGE(TAG, "Failed to initialize HTTP client");
        return -1;
    }
    
    // Set method to GET
    esp_http_client_set_method(client, HTTP_METHOD_GET);
    
    // Check if file exists for resume
    FILE *fp = fopen(save_path, "rb");
    size_t resume_pos = 0;
    if (fp != NULL) {
        fseek(fp, 0, SEEK_END);
        resume_pos = ftell(fp);
        fclose(fp);
        
        if (resume_pos > 0) {
            char range_header[64];
            snprintf(range_header, sizeof(range_header), "bytes=%zu-", resume_pos);
            esp_http_client_set_header(client, "Range", range_header);
            ESP_LOGI(TAG, "Resuming download from byte %zu", resume_pos);
        }
    }
    
    // Open file for writing
    fp = fopen(save_path, resume_pos > 0 ? "ab" : "wb");
    if (fp == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        esp_http_client_cleanup(client);
        return -1;
    }
    
    // Perform request
    esp_err_t err = esp_http_client_open(client, 0);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to open HTTP connection: %s", esp_err_to_name(err));
        fclose(fp);
        esp_http_client_cleanup(client);
        return -1;
    }
    
    // Read content length
    int content_length = esp_http_client_fetch_headers(client);
    int status_code = esp_http_client_get_status_code(client);
    
    ESP_LOGI(TAG, "HTTP Status: %d, Content-Length: %d", status_code, content_length);
    
    if (status_code != 200 && status_code != 206) {
        ESP_LOGE(TAG, "HTTP request failed with status: %d", status_code);
        fclose(fp);
        esp_http_client_close(client);
        esp_http_client_cleanup(client);
        return -1;
    }
    
    // Read and write firmware data
    char buffer[FIRMWARE_BUFFER_SIZE];
    int total_read = 0;
    int read_len;
    
    while ((read_len = esp_http_client_read(client, buffer, sizeof(buffer))) > 0) {
        size_t written = fwrite(buffer, 1, read_len, fp);
        if (written != read_len) {
            ESP_LOGE(TAG, "Failed to write to file");
            fclose(fp);
            esp_http_client_close(client);
            esp_http_client_cleanup(client);
            return -1;
        }
        
        total_read += read_len;
        
        // Progress update (optional)
        if (content_length > 0) {
            int progress = (total_read * 100) / content_length;
            ESP_LOGI(TAG, "Download progress: %d%% (%d/%d bytes)", 
                     progress, total_read, content_length);
        }
        
        // Check if download exceeds maximum size
        if (total_read > MAX_FIRMWARE_SIZE) {
            ESP_LOGE(TAG, "Firmware size exceeds maximum");
            fclose(fp);
            esp_http_client_close(client);
            esp_http_client_cleanup(client);
            return -1;
        }
    }
    
    fclose(fp);
    esp_http_client_close(client);
    esp_http_client_cleanup(client);
    
    ESP_LOGI(TAG, "Firmware downloaded successfully: %d bytes", total_read);
    return 0;
}
```

### Download with Progress Callback

```c
typedef struct {
    size_t downloaded;
    size_t total;
    int percentage;
} download_progress_t;

// Progress callback function
static int download_progress_cb(esp_http_client_event_t *evt) {
    switch (evt->event_id) {
        case HTTP_EVENT_ON_DATA:
            if (evt->data_len > 0) {
                download_progress_t *progress = (download_progress_t *)evt->user_data;
                progress->downloaded += evt->data_len;
                
                if (progress->total > 0) {
                    progress->percentage = (progress->downloaded * 100) / progress->total;
                    ESP_LOGI(TAG, "Download progress: %d%%", progress->percentage);
                }
            }
            break;
        default:
            break;
    }
    return ESP_OK;
}

int download_firmware_with_progress(const firmware_update_info_t *update_info,
                                    const char *save_path,
                                    download_progress_t *progress) {
    esp_http_client_config_t config = {
        .url = update_info->download_url,
        .timeout_ms = 300000, // 5 minutes
        .event_handler = download_progress_cb,
        .user_data = progress,
    };
    
    esp_http_client_handle_t client = esp_http_client_init(&config);
    esp_http_client_set_method(client, HTTP_METHOD_GET);
    
    // ... rest of download logic
}
```

## Step 4: Verify Firmware

Verify firmware SHA256 hash before installation.

### SHA256 Verification Function

```c
#include "mbedtls/sha256.h"
#include <stdio.h>

/**
 * Calculate SHA256 hash of file
 */
int calculate_file_sha256(const char *file_path, char *sha256_str) {
    FILE *fp = fopen(file_path, "rb");
    if (fp == NULL) {
        ESP_LOGE(TAG, "Failed to open file for SHA256 calculation");
        return -1;
    }
    
    mbedtls_sha256_context sha256_ctx;
    mbedtls_sha256_init(&sha256_ctx);
    mbedtls_sha256_starts(&sha256_ctx, 0); // 0 = SHA256, 1 = SHA224
    
    unsigned char hash[32];
    char buffer[FIRMWARE_BUFFER_SIZE];
    size_t bytes_read;
    
    while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
        mbedtls_sha256_update(&sha256_ctx, (unsigned char *)buffer, bytes_read);
    }
    
    mbedtls_sha256_finish(&sha256_ctx, hash);
    mbedtls_sha256_free(&sha256_ctx);
    fclose(fp);
    
    // Convert hash to hex string
    for (int i = 0; i < 32; i++) {
        sprintf(sha256_str + (i * 2), "%02x", hash[i]);
    }
    sha256_str[64] = '\0';
    
    return 0;
}

/**
 * Verify firmware SHA256 hash
 * Returns 0 if hash matches, negative on error
 */
int verify_firmware_hash(const char *file_path, const char *expected_sha256) {
    char calculated_sha256[65];
    
    ESP_LOGI(TAG, "Calculating firmware SHA256 hash...");
    
    if (calculate_file_sha256(file_path, calculated_sha256) != 0) {
        ESP_LOGE(TAG, "Failed to calculate SHA256 hash");
        return -1;
    }
    
    ESP_LOGI(TAG, "Expected SHA256: %s", expected_sha256);
    ESP_LOGI(TAG, "Calculated SHA256: %s", calculated_sha256);
    
    // Compare hashes (case-insensitive)
    if (strcasecmp(calculated_sha256, expected_sha256) != 0) {
        ESP_LOGE(TAG, "SHA256 hash mismatch!");
        return -1;
    }
    
    ESP_LOGI(TAG, "Firmware SHA256 verification passed");
    return 0;
}
```

## Step 5: Complete Workflow

Complete firmware update workflow combining all steps.

### Main Update Function

```c
#define FIRMWARE_PATH "/spiffs/firmware.bin"
#define TEMP_FIRMWARE_PATH "/spiffs/firmware_temp.bin"

/**
 * Complete firmware update workflow
 */
int perform_firmware_update(void) {
    firmware_update_info_t update_info = {0};
    int ret;
    
    // Step 1: Check for updates
    ESP_LOGI(TAG, "=== Step 1: Checking for updates ===");
    ret = check_for_update(DEVICE_ID, CURRENT_VERSION, &update_info);
    if (ret != 0) {
        ESP_LOGE(TAG, "Failed to check for updates");
        return -1;
    }
    
    if (!update_info.update_available) {
        ESP_LOGI(TAG, "No update available. Current version: %s", CURRENT_VERSION);
        return 0;
    }
    
    // Step 2: Check device capabilities
    ESP_LOGI(TAG, "=== Step 2: Verifying device capabilities ===");
    if (get_battery_level() < update_info.min_battery) {
        ESP_LOGW(TAG, "Battery level too low. Minimum required: %d%%, Current: %d%%",
                 update_info.min_battery, get_battery_level());
        return -1;
    }
    
    if (get_signal_strength() < update_info.min_signal) {
        ESP_LOGW(TAG, "Signal strength too low. Minimum required: %d, Current: %d",
                 update_info.min_signal, get_signal_strength());
        return -1;
    }
    
    // Step 3: Download firmware
    ESP_LOGI(TAG, "=== Step 3: Downloading firmware ===");
    ESP_LOGI(TAG, "Firmware: %s v%s", update_info.firmware_id, update_info.version);
    ESP_LOGI(TAG, "Size: %lld bytes", update_info.size);
    
    ret = download_firmware(&update_info, TEMP_FIRMWARE_PATH);
    if (ret != 0) {
        ESP_LOGE(TAG, "Failed to download firmware");
        return -1;
    }
    
    // Step 4: Verify firmware
    ESP_LOGI(TAG, "=== Step 4: Verifying firmware ===");
    ret = verify_firmware_hash(TEMP_FIRMWARE_PATH, update_info.sha256);
    if (ret != 0) {
        ESP_LOGE(TAG, "Firmware verification failed");
        remove(TEMP_FIRMWARE_PATH); // Clean up
        return -1;
    }
    
    // Step 5: Install firmware (device-specific)
    ESP_LOGI(TAG, "=== Step 5: Installing firmware ===");
    ret = install_firmware(TEMP_FIRMWARE_PATH);
    if (ret != 0) {
        ESP_LOGE(TAG, "Failed to install firmware");
        remove(TEMP_FIRMWARE_PATH);
        return -1;
    }
    
    // Step 6: Report success
    ESP_LOGI(TAG, "=== Step 6: Reporting update success ===");
    report_update_success(update_info.campaign_id, update_info.version);
    
    ESP_LOGI(TAG, "Firmware update completed successfully!");
    return 0;
}

/**
 * Device-specific firmware installation
 * This is platform-dependent and needs to be implemented based on your hardware
 */
int install_firmware(const char *firmware_path) {
    // ESP32 example: Use esp_ota_handle_t
    #ifdef ESP_PLATFORM
        // ESP32 OTA installation code
        // See ESP-IDF OTA examples
    #else
        // For other platforms:
        // 1. Copy firmware to flash partition
        // 2. Verify installation
        // 3. Set boot flags
    #endif
    
    return 0;
}

/**
 * Report update success to server
 */
int report_update_success(const char *campaign_id, const char *new_version) {
    char url[256];
    char request_body[512];
    
    snprintf(url, sizeof(url), "%s/devices/%s/version", API_BASE_URL, DEVICE_ID);
    snprintf(request_body, sizeof(request_body),
        "{"
        "\"campaign_id\":\"%s\","
        "\"old_version\":\"%s\","
        "\"new_version\":\"%s\","
        "\"status\":\"success\""
        "}",
        campaign_id, CURRENT_VERSION, new_version);
    
    esp_http_client_handle_t client = http_client_init(url);
    esp_http_client_set_method(client, HTTP_METHOD_POST);
    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_post_field(client, request_body, strlen(request_body));
    
    esp_err_t err = esp_http_client_perform(client);
    int status_code = esp_http_client_get_status_code(client);
    esp_http_client_cleanup(client);
    
    if (err != ESP_OK || status_code != 200) {
        ESP_LOGE(TAG, "Failed to report update success");
        return -1;
    }
    
    ESP_LOGI(TAG, "Update success reported to server");
    return 0;
}
```

### Periodic Update Check

```c
void firmware_update_task(void *pvParameters) {
    const TickType_t delay = pdMS_TO_TICKS(3600000); // Check every hour
    
    while (1) {
        // Only check for updates if connected to WiFi
        if (wifi_is_connected()) {
            perform_firmware_update();
        } else {
            ESP_LOGW(TAG, "WiFi not connected, skipping update check");
        }
        
        vTaskDelay(delay);
    }
}

// Start update check task
void start_firmware_update_task(void) {
    xTaskCreate(
        firmware_update_task,
        "firmware_update",
        8192,
        NULL,
        5,
        NULL
    );
}
```

## Error Handling

### Common Errors and Solutions

```c
typedef enum {
    UPDATE_ERROR_NONE = 0,
    UPDATE_ERROR_NETWORK,
    UPDATE_ERROR_HTTP,
    UPDATE_ERROR_DOWNLOAD,
    UPDATE_ERROR_VERIFICATION,
    UPDATE_ERROR_INSTALLATION,
    UPDATE_ERROR_STORAGE,
} update_error_t;

const char* update_error_to_string(update_error_t error) {
    switch (error) {
        case UPDATE_ERROR_NONE:
            return "No error";
        case UPDATE_ERROR_NETWORK:
            return "Network connection error";
        case UPDATE_ERROR_HTTP:
            return "HTTP request error";
        case UPDATE_ERROR_DOWNLOAD:
            return "Firmware download error";
        case UPDATE_ERROR_VERIFICATION:
            return "Firmware verification error";
        case UPDATE_ERROR_INSTALLATION:
            return "Firmware installation error";
        case UPDATE_ERROR_STORAGE:
            return "Storage error";
        default:
            return "Unknown error";
    }
}

int perform_firmware_update_with_retry(void) {
    const int max_retries = 3;
    int retry_count = 0;
    
    while (retry_count < max_retries) {
        int ret = perform_firmware_update();
        if (ret == 0) {
            return 0; // Success
        }
        
        retry_count++;
        ESP_LOGW(TAG, "Update failed, retrying (%d/%d)...", retry_count, max_retries);
        vTaskDelay(pdMS_TO_TICKS(5000)); // Wait 5 seconds before retry
    }
    
    ESP_LOGE(TAG, "Firmware update failed after %d retries", max_retries);
    return -1;
}
```

## Best Practices

### 1. Check Before Download

```c
int check_device_ready_for_update(const firmware_update_info_t *update_info) {
    // Check battery level
    int battery_level = get_battery_level();
    if (battery_level < update_info->min_battery) {
        ESP_LOGW(TAG, "Battery too low: %d%% (minimum: %d%%)", 
                 battery_level, update_info->min_battery);
        return -1;
    }
    
    // Check signal strength
    int signal_strength = get_signal_strength();
    if (signal_strength < update_info->min_signal) {
        ESP_LOGW(TAG, "Signal too weak: %d (minimum: %d)", 
                 signal_strength, update_info->min_signal);
        return -1;
    }
    
    // Check available storage
    size_t free_storage = get_free_storage();
    if (free_storage < update_info->size) {
        ESP_LOGW(TAG, "Insufficient storage: %zu bytes (required: %lld)", 
                 free_storage, update_info->size);
        return -1;
    }
    
    return 0;
}
```

### 2. Implement Resume Support

The download function already includes resume support. Ensure your file system supports append mode.

### 3. Progress Reporting

Report download progress to the server:

```c
int report_progress(const char *campaign_id, const char *status, int percentage) {
    char url[256];
    char request_body[512];
    
    snprintf(url, sizeof(url), "%s/devices/%s/progress", API_BASE_URL, DEVICE_ID);
    snprintf(request_body, sizeof(request_body),
        "{"
        "\"campaign_id\":\"%s\","
        "\"status\":\"%s\","
        "\"progress\":{"
        "\"percentage\":%d"
        "}"
        "}",
        campaign_id, status, percentage);
    
    // Send progress report
    // ... HTTP POST implementation
    
    return 0;
}
```

### 4. Rollback Capability

Implement rollback mechanism for failed updates:

```c
int rollback_firmware(void) {
    ESP_LOGW(TAG, "Rolling back to previous firmware version...");
    // Platform-specific rollback implementation
    return 0;
}
```

## Complete Example

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "wifi_connect.h"

static const char *TAG = "MAIN";

void app_main(void) {
    ESP_LOGI(TAG, "Starting IoT Device Firmware Update System");
    
    // Initialize NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);
    
    // Connect to WiFi
    wifi_init_sta();
    
    // Wait for WiFi connection
    while (!wifi_is_connected()) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    
    ESP_LOGI(TAG, "WiFi connected, starting firmware update check");
    
    // Perform initial firmware update check
    perform_firmware_update_with_retry();
    
    // Start periodic update check task
    start_firmware_update_task();
    
    // Continue with your main application logic
    while (1) {
        // Your application code here
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## Testing

### Test Update Check

```c
void test_update_check(void) {
    firmware_update_info_t update_info = {0};
    
    ESP_LOGI(TAG, "Testing update check...");
    
    int ret = check_for_update(DEVICE_ID, CURRENT_VERSION, &update_info);
    assert(ret == 0);
    
    if (update_info.update_available) {
        ESP_LOGI(TAG, "Update available: %s", update_info.version);
        ESP_LOGI(TAG, "Download URL: %s", update_info.download_url);
    } else {
        ESP_LOGI(TAG, "No update available");
    }
}
```

## References

- [ESP-IDF HTTP Client Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/protocols/esp_http_client.html)
- [ESP-IDF OTA Updates](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html)
- [cJSON Library](https://github.com/DaveGamble/cJSON)
- [OTAP API Documentation](./firmware-download-guide.md)

## Support

For issues or questions:
- Check logs for error messages
- Verify network connectivity
- Ensure API endpoints are correct
- Verify firmware signature and SHA256 hash
- Check device capabilities (battery, signal, storage)

