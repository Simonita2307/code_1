#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "pico/time.h"
#include "lcd1602.h"

// GPIO пинове
#define BUTTON_START  2
#define BUTTON_STOP   3
#define BUTTON_RESET  4
#define BUTTON_CLEAR  5

// LCD I2C настройки
#define I2C_SDA  0
#define I2C_SCL  1
#define I2C_ADDR 0x27

bool running = false;
absolute_time_t start_time;
int elapsed_ms = 0;

void init_buttons() {
    gpio_init(BUTTON_START);
    gpio_init(BUTTON_STOP);
    gpio_init(BUTTON_RESET);
    gpio_init(BUTTON_CLEAR);

    gpio_set_dir(BUTTON_START, GPIO_IN);
    gpio_set_dir(BUTTON_STOP, GPIO_IN);
    gpio_set_dir(BUTTON_RESET, GPIO_IN);
    gpio_set_dir(BUTTON_CLEAR, GPIO_IN);

    gpio_pull_up(BUTTON_START);
    gpio_pull_up(BUTTON_STOP);
    gpio_pull_up(BUTTON_RESET);
    gpio_pull_up(BUTTON_CLEAR);
}

void display_time(int ms) {
    int seconds = ms / 1000;
    int minutes = seconds / 60;
    seconds %= 60;

    char buffer[17];
    snprintf(buffer, sizeof(buffer), "Time: %02d:%02d", minutes, seconds);
    lcd_set_cursor(0, 0);
    lcd_print(buffer);
}

int main() {
    stdio_init_all();

    // LCD инициализация
    lcd_init(I2C_SDA, I2C_SCL, I2C_ADDR);
    sleep_ms(100); // пауза за стабилизиране
    lcd_clear();
    lcd_set_cursor(0, 0);
    lcd_print("RPI Pico Timer");

    init_buttons();

    while (true) {
        // Старт
        if (!gpio_get(BUTTON_START) && !running) {
            running = true;
            start_time = get_absolute_time();
        }

        // Стоп
        if (!gpio_get(BUTTON_STOP) && running) {
            running = false;
            elapsed_ms += absolute_time_diff_us(start_time, get_absolute_time()) / 1000;
        }

        // Рестарт (нулиране на времето)
        if (!gpio_get(BUTTON_RESET)) {
            running = false;
            elapsed_ms = 0;
        }

        // Изчистване на дисплея
        if (!gpio_get(BUTTON_CLEAR)) {
            lcd_clear();
            lcd_set_cursor(0, 1);
            lcd_print("Cleared!");
        }

        // Обновяване на дисплея
        if (running) {
            int current = absolute_time_diff_us(start_time, get_absolute_time()) / 1000;
            display_time(elapsed_ms + current);
        } else {
            display_time(elapsed_ms);
        }

        sleep_ms(200); // debounce
    }

    return 0;
}
