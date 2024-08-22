# Bài báo cáo training




## Giải thích code mẫu Wifi Station
## Khai báo thư viện 
```bash 
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "lwip/err.h"
#include "lwip/sys.h"
```
- string.h: Đây là thư viện chuẩn của C cung cấp các hàm để xử lý chuỗi như sao chép, nối và so sánh chuỗi.

- freertos/FreeRTOS.h: Đây là thư viện của FreeRTOS, một hệ điều hành thời gian thực được sử dụng để quản lý đa nhiệm, cung cấp các API để tạo và quản lý task, queue, semaphore, và các đối tượng đồng bộ hóa khác.

- freertos/task.h: Thư viện này cung cấp các hàm cụ thể để làm việc với các task trong FreeRTOS, bao gồm tạo task, xóa task, và chuyển đổi giữa các task.

- freertos/event_groups.h: Thư viện này cho phép bạn sử dụng các nhóm sự kiện (event groups) trong FreeRTOS, hữu ích trong việc đồng bộ hóa giữa các task hoặc giữa task và interrupt.

- esp_system.h: Thư viện này cung cấp các API liên quan đến hệ thống của ESP-IDF, bao gồm khởi động lại hệ thống, lấy thông tin hệ thống, và các thao tác quản lý nguồn.

- esp_wifi.h: Đây là thư viện dành riêng cho ESP32 để quản lý Wi-Fi, bao gồm khởi tạo Wi-Fi, kết nối vào mạng, và xử lý các sự kiện liên quan đến Wi-Fi.

- esp_event.h: Thư viện này cung cấp một hệ thống quản lý sự kiện cho ESP-IDF, cho phép đăng ký và xử lý các sự kiện như Wi-Fi, hệ thống, và sự kiện từ các thành phần khác.

- esp_log.h: Thư viện này cung cấp các hàm để ghi log, giúp theo dõi và gỡ lỗi các ứng dụng trên ESP32.

- nvs_flash.h: Thư viện này cho phép truy cập và quản lý bộ nhớ NVS (Non-Volatile Storage), nơi bạn có thể lưu trữ dữ liệu không mất đi khi mất nguồn.

- lwip/err.h và lwip/sys.h: Đây là các thư viện thuộc phần lwIP (Lightweight IP), một stack giao thức mạng được sử dụng trong ESP32 để xử lý các kết nối mạng TCP/IP. err.h chứa các định nghĩa mã lỗi, và sys.h chứa các hàm và macro cần thiết cho việc tích hợp với hệ điều hành. 

## Đặt lại cấu hình Wifi 
```bash 
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
// điều chỉnh tên Wifi
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
// điều chỉnh mật khẩu Wifi
#define EXAMPLE_ESP_MAXIMUM_RETRY  CONFIG_ESP_MAXIMUM_RETRY
// xác định số lần thử lại kết nối Wi-Fi nếu kết nối bị thất bại
```


```bash 
static EventGroupHandle_t s_wifi_event_group;
// Dòng mã static EventGroupHandle_t s_wifi_event_group; 
khai báo một biến tĩnh (static) có tên là s_wifi_event_group, được sử dụng để lưu trữ handle của một event group trong 
FreeRTOS.
//Trong các ứng dụng ESP32, s_wifi_event_group thường được sử dụng để đồng bộ hóa các sự kiện liên quan đến Wi-Fi, như kết nối hoặc ngắt kết nối, chờ IP được gán từ DHCP, v.v. Các bit trong event group có thể được dùng để theo dõi trạng thái của các sự kiện này.

``` 

```bash 
static const char *TAG = "wifi station";
//Dòng mã static const char *TAG = "wifi station"; khai báo một biến con trỏ chuỗi TAG với giá trị là "wifi station".
//Logging: Biến TAG thường được sử dụng với hệ thống ghi log (logging) trong ESP-IDF. ESP-IDF cung cấp một số hàm logging như ESP_LOGI, ESP_LOGW, ESP_LOGE, v.v. để in thông tin, cảnh báo, và lỗi. Biến TAG được truyền vào các hàm này để cho biết nguồn gốc của thông điệp log, giúp dễ dàng theo dõi và gỡ lỗi.
```
```bash 
static int s_retry_num = 0; 
//Khai báo số lần thử lại khi kết nối
```

```bash 
static void event_handler(void* arg, esp_event_base_t event_base,
                                int32_t event_id, void* event_data)
 // Đăng ký một callback cho một loại sự kiện cụ thể. Khi sự kiện
 đó xảy ra, callback này sẽ được gọi.   
 // event_base: xác định nhóm sự kiện
 // event_id: Xác định cụ thể sự kiên trong nhóm                              
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
    
        esp_wifi_connect();
//Khi sự kiện Wi-Fi bắt đầu (WIFI_EVENT_STA_START), hàm sẽ gọi esp_wifi_connect() để kết nối vào Access Point.
Khi Wi-Fi bị ngắt kết nối (WIFI_EVENT_STA_DISCONNECTED), hàm cũng gọi esp_wifi_connect() để tự động thử kết nối lại.

    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
    //Khi Wi-Fi bị ngắt kết nối (WIFI_EVENT_STA_DISCONNECTED), hàm cũng gọi esp_wifi_connect() để tự động thử kết nối lại.

        if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY) {
            esp_wifi_connect();
            s_retry_num++;// tăng biến đếm số lần thử lại
            ESP_LOGI(TAG, "retry to connect to the AP");// in ra monitor
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);//set bit trong 1 nhóm sự kiện đang cho xử lí  
        }
        ESP_LOGI(TAG,"connect to the AP fail");// in ra kết nối fail
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {// Nếu kết nối được 
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));// in ra IP
        s_retry_num = 0;// gán số lần thử lại bằng 0
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    //set bit trong 1 nhóm sự kiện đang cho xử lí
    }
}
```
