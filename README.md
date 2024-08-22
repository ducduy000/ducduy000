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
//Trong các ứng dụng ESP32, s_wifi_event_group thường được sử dụng để đồng bộ hóa các sự kiện liên quan đến Wi-Fi,
 như kết nối hoặc ngắt kết nối, chờ IP được gán từ DHCP, v.v.
Các bit trong event group có thể được dùng để theo dõi trạng thái của các sự kiện này.

``` 

```bash 
static const char *TAG = "wifi station";
//Dòng mã static const char *TAG = "wifi station"; khai báo một biến con trỏ chuỗi TAG với giá trị là "wifi station".
//Logging: Biến TAG thường được sử dụng với hệ thống ghi log (logging) trong ESP-IDF. ESP-IDF cung cấp
một số hàm logging như ESP_LOGI, ESP_LOGW, ESP_LOGE, v.v. để in thông tin, cảnh báo, và lỗi.
Biến TAG được truyền vào các hàm này để cho biết nguồn gốc của thông điệp log, giúp dễ dàng theo dõi và gỡ lỗi.
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
```bash 
void wifi_init_sta(void) // khởi tạo wifi ở chế độ station
{
    s_wifi_event_group = xEventGroupCreate(); // tạo freeRTOS nhóm sự kiện để quản lí trạng thái kết nối
    // hàm ESP_ERROR_CHECK check lỗi hoặc không lỗi, nếu nó kiểm tra mà không trả về  
    // ESP_OK thì xác nhận và ghi lại thông tin đã xác nhận
    ESP_ERROR_CHECK(esp_netif_init()); // khởi tạo lwip: giao thức quản lí tất cả liên quan đến kết nối mạng
    // là ngăn xếp TCP/IP, được dùng để thực hiện các giao thức khác nhau là TCP UDP DHCP etc

    ESP_ERROR_CHECK(esp_event_loop_create_default());// tạo một vòng lặp sự kiện để cho hệ thống gửi các sự kiện về
tác vụ sự kiện
    esp_netif_create_default_wifi_sta(); // tạo giao diện mạng wifi mặc định chế độ station
    // 2 dòng dưới để khởi tạo wifi rồi cung cấp dữ liệu cho wifi driver, trách nhiệm là khởi tạo tác vụ wifi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT(); // cấu hình khởi tạo wifi mặc định
    ESP_ERROR_CHECK(esp_wifi_init(&cfg)); // khởi tạo wifi với cấu hình mặc định
    
    esp_event_handler_instance_t instance_any_id; // Handle instance_any_id giúp bạn quản lý việc đăng ký
và hủy đăng ký event handler một cách rõ ràng và có tổ chức. 
    esp_event_handler_instance_t instance_got_ip; //  Biến này được sử dụng để lưu trữ handle của một event handler đã đăng ký,
cụ thể là event handler sẽ xử lý sự kiện liên quan đến việc thiết bị nhận được địa chỉ IP 
    // 2 hàm dưới để đăng kí các sự kiện cần xử lí, như wifi events và ip events
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT,
                                                        IP_EVENT_STA_GOT_IP,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_got_ip));
wifi_config_t wifi_config = { // đăng nhập Wifi cho ESP32
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
            /* Authmode threshold resets to WPA2 as default if password matches WPA2 standards (password phải đạt tiêu chuẩn WPA2).
             * If you want to connect the device to deprecated WEP/WPA networks, Please set the threshold value
             * to WIFI_AUTH_WEP/WIFI_AUTH_WPA_PSK and set the password with length and format matching to
             * WIFI_AUTH_WEP/WIFI_AUTH_WPA_PSK standards.
             */
            .threshold.authmode = ESP_WIFI_SCAN_AUTH_MODE_THRESHOLD,
            .sae_pwe_h2e = ESP_WIFI_SAE_MODE,
            .sae_h2e_identifier = EXAMPLE_H2E_IDENTIFIER,
        },
    };

ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA) ); // thực hiện việc cài đặt chế độ hoạt động của Wi-Fi trên ESP32 và
kiểm tra lỗi để đảm bảo rằng việc cài đặt này thành công.
    // 2 dòng dưới gán tài khoản mật khẩu wifi, làm wifi driver được cấu hình với thông tin đăng nhập mạng và wifi mode
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config) );
    // khởi tạo sau khi đã được cấu hình chỉ định
    ESP_ERROR_CHECK(esp_wifi_start() );
    // hoàn thành khởi tạo chế độ wifi station, in ra
    ESP_LOGI(TAG, "wifi_init_sta finished.");


    /* Waiting until either the connection is established (WIFI_CONNECTED_BIT) or connection failed for the maximum
     * number of re-tries (WIFI_FAIL_BIT). The bits are set by event_handler() (see above) */
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            portMAX_DELAY); // hàm này kiểm tra kết nối được hay không được thì in ra cả passs và id
            // không được thì cũng in ra cả pass va id

    /* xEventGroupWaitBits() returns the bits before the call returned, hence we can test which event actually
     * happened. */
    if (bits & WIFI_CONNECTED_BIT) { // Nếu thiết bị đã kết nối thành công,
ESP32 sẽ ghi log thông tin về SSID và mật khẩu của mạng Wi-Fi mà nó đã kết nối.
        ESP_LOGI(TAG, "connected to ap SSID:%s password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else if (bits & WIFI_FAIL_BIT) { // tương tự như trên
        ESP_LOGI(TAG, "Failed to connect to SSID:%s, password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else {
        ESP_LOGE(TAG, "UNEXPECTED EVENT"); // không phải 2 trường hợp trên
thì in ra thông báo về sự kiên bất thường
    }
}

void app_main(void)
{
    //Initialize NVS 
    // NVS là 1 hệ thống lưu trữ tạm thời, cho phép bạn lưu trữ dữ liệu trên flash bộ nhớ của esp 32 không bị mất khi tắt thiết bị
    esp_err_t ret = nvs_flash_init(); // khởi tạo hệ thống lưu trữ không thay đổi
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
      ESP_ERROR_CHECK(nvs_flash_erase()); //nếu không có trong hoặc đã có version khác thì xóa
      ret = nvs_flash_init(); //tạo lại NVS mới
    }
    ESP_ERROR_CHECK(ret); // check lỗi

    ESP_LOGI(TAG, "ESP_WIFI_MODE_STA"); // ghi nhật kí (lỗi hoặc thông tin)
    wifi_init_sta();
}
```
