/**
 * nvgpu-mon - Lightweight NVIDIA GPU monitoring utility
 * 
 * Compilation:
 *   gcc -O2 -Wall -Wextra -o nvgpu-mon nvgpu-mon.c -lnvidia-ml
 * 
 * Usage:
 *   ./nvgpu-mon [interval_seconds]
 *   Default interval: 1 second
 * 
 * ANSI escape sequences for terminal dashboard refresh
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <signal.h>
#include <stdarg.h>

#include <nvml.h>

/* Constants */
#define MAX_DEVICES 64
#define BUFFER_SIZE 256
#define DASHBOARD_REFRESH "\033[H\033[J"
#define CURSOR_HOME "\033[H"

/* Global state */
static volatile sig_atomic_t running = 1;
static int interval = 1;
static nvmlDevice_t devices[MAX_DEVICES];
static unsigned int device_count = 0;

/* Secure printf wrapper */
static void safe_printf(const char *format, ...) {
    va_list args;
    va_start(args, format);
    vprintf(format, args);
    va_end(args);
}

/* Signal handler */
static void signal_handler(int sig) {
    (void)sig;
    running = 0;
}

/* Initialize NVML */
static int nvml_init(void) {
    nvmlReturn_t result;
    
    result = nvmlInit();
    if (result != NVML_SUCCESS) {
        fprintf(stderr, "Failed to initialize NVML: %s\n", nvmlErrorString(result));
        return -1;
    }
    
    result = nvmlDeviceGetCount(&device_count);
    if (result != NVML_SUCCESS) {
        fprintf(stderr, "Failed to get device count: %s\n", nvmlErrorString(result));
        nvmlShutdown();
        return -1;
    }
    
    if (device_count == 0) {
        fprintf(stderr, "No NVIDIA devices found\n");
        nvmlShutdown();
        return -1;
    }
    
    if (device_count > MAX_DEVICES) {
        device_count = MAX_DEVICES;
    }
    
    for (unsigned int i = 0; i < device_count; i++) {
        result = nvmlDeviceGetHandleByIndex(i, &devices[i]);
        if (result != NVML_SUCCESS) {
            fprintf(stderr, "Failed to get device %u handle: %s\n", i, nvmlErrorString(result));
            nvmlShutdown();
            return -1;
        }
    }
    
    return 0;
}

/* Clean shutdown */
static void nvml_cleanup(void) {
    nvmlShutdown();
}

/* Format memory size */
static void format_memory(unsigned long long bytes, char *buffer, size_t size) {
    const char *units[] = {"B", "KiB", "MiB", "GiB", "TiB"};
    int unit_index = 0;
    double value = (double)bytes;
    
    while (value >= 1024.0 && unit_index < 4) {
        value /= 1024.0;
        unit_index++;
    }
    
    snprintf(buffer, size, "%.2f %s", value, units[unit_index]);
}

/* Get GPU metrics */
static int get_gpu_metrics(unsigned int device_index, char *output, size_t output_size) {
    nvmlDevice_t device = devices[device_index];
    nvmlReturn_t result;
    char device_name[NVML_DEVICE_NAME_BUFFER_SIZE];
    char temp_str[32], fan_str[32], util_str[32];
    char mem_total_str[32], mem_free_str[32], mem_used_str[32];
    char clock_core_str[32], clock_mem_str[32];
    unsigned int temp, fan;
    unsigned long long mem_total, mem_free, mem_used;
    unsigned int clock_core, clock_mem;
    size_t pos = 0;
    int written;
    
    /* Get device name */
    result = nvmlDeviceGetName(device, device_name, sizeof(device_name));
    if (result != NVML_SUCCESS) {
        snprintf(device_name, sizeof(device_name), "GPU %u", device_index);
    }
    
    /* Get temperature */
    result = nvmlDeviceGetTemperature(device, NVML_TEMPERATURE_GPU, &temp);
    if (result == NVML_SUCCESS) {
        snprintf(temp_str, sizeof(temp_str), "%u°C", temp);
    } else {
        snprintf(temp_str, sizeof(temp_str), "N/A");
    }
    
    /* Get fan speed */
    result = nvmlDeviceGetFanSpeed(device, &fan);
    if (result == NVML_SUCCESS) {
        snprintf(fan_str, sizeof(fan_str), "%u%%", fan);
    } else {
        snprintf(fan_str, sizeof(fan_str), "N/A");
    }
    
    /* Get utilization */
    nvmlUtilization_t utilization;
    result = nvmlDeviceGetUtilizationRates(device, &utilization);
    if (result == NVML_SUCCESS) {
        snprintf(util_str, sizeof(util_str), "%u%%", utilization.gpu);
    } else {
        snprintf(util_str, sizeof(util_str), "N/A");
    }
    
    /* Get memory info */
    nvmlMemory_t memory;
    result = nvmlDeviceGetMemoryInfo(device, &memory);
    if (result == NVML_SUCCESS) {
        mem_total = memory.total;
        mem_free = memory.free;
        mem_used = memory.used;
        format_memory(mem_total, mem_total_str, sizeof(mem_total_str));
        format_memory(mem_free, mem_free_str, sizeof(mem_free_str));
        format_memory(mem_used, mem_used_str, sizeof(mem_used_str));
    } else {
        snprintf(mem_total_str, sizeof(mem_total_str), "N/A");
        snprintf(mem_free_str, sizeof(mem_free_str), "N/A");
        snprintf(mem_used_str, sizeof(mem_used_str), "N/A");
    }
    
    /* Get clock speeds */
    result = nvmlDeviceGetClockInfo(device, NVML_CLOCK_GRAPHICS, &clock_core);
    if (result == NVML_SUCCESS) {
        snprintf(clock_core_str, sizeof(clock_core_str), "%u MHz", clock_core);
    } else {
        snprintf(clock_core_str, sizeof(clock_core_str), "N/A");
    }
    
    result = nvmlDeviceGetClockInfo(device, NVML_CLOCK_MEM, &clock_mem);
    if (result == NVML_SUCCESS) {
        snprintf(clock_mem_str, sizeof(clock_mem_str), "%u MHz", clock_mem);
    } else {
        snprintf(clock_mem_str, sizeof(clock_mem_str), "N/A");
    }
    
    /* Build output - compact dashboard format */
    written = snprintf(output + pos, output_size - pos,
        "┌─ %s\n"
        "│ Temp: %s  Fan: %s  Util: %s\n"
        "│ Mem: %s / %s (Free: %s)\n"
        "│ Core: %s  Memory: %s\n"
        "└─────────────────────────────────\n",
        device_name,
        temp_str, fan_str, util_str,
        mem_used_str, mem_total_str, mem_free_str,
        clock_core_str, clock_mem_str);
    
    if (written < 0 || (size_t)written >= output_size - pos) {
        return -1;
    }
    pos += written;
    
    return 0;
}

/* Print dashboard header */
static void print_header(void) {
    time_t now;
    struct tm *tm_info;
    char time_buffer[64];
    
    time(&now);
    tm_info = localtime(&now);
    strftime(time_buffer, sizeof(time_buffer), "%Y-%m-%d %H:%M:%S", tm_info);
    
    safe_printf("\n");
    safe_printf("╔══════════════════════════════════════════════════╗\n");
    safe_printf("║  NVIDIA GPU Monitor  │  %-30s ║\n", time_buffer);
    safe_printf("╚══════════════════════════════════════════════════╝\n");
    safe_printf("\n");
}

/* Main monitoring loop */
static void monitor_loop(void) {
    char output_buffer[4096];
    
    while (running) {
        /* Clear screen and move cursor home */
        safe_printf(DASHBOARD_REFRESH);
        
        print_header();
        
        for (unsigned int i = 0; i < device_count && running; i++) {
            if (get_gpu_metrics(i, output_buffer, sizeof(output_buffer)) == 0) {
                safe_printf("%s", output_buffer);
            } else {
                safe_printf("Error reading GPU %u metrics\n", i);
            }
        }
        
        safe_printf("\nPress Ctrl+C to exit\n");
        fflush(stdout);
        
        /* Sleep for interval, checking for signals */
        for (int i = 0; i < interval && running; i++) {
            nanosleep(&(struct timespec){.tv_sec = 1, .tv_nsec = 0}, NULL);
        }
    }
}

/* Main entry point */
int main(int argc, char **argv) {
    /* Parse command line */
    if (argc > 1) {
        interval = atoi(argv[1]);
        if (interval < 1) {
            interval = 1;
        }
    }
    
    /* Setup signal handlers */
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = signal_handler;
    sigaction(SIGINT, &sa, NULL);
    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGQUIT, &sa, NULL);
    
    /* Initialize NVML */
    if (nvml_init() != 0) {
        return EXIT_FAILURE;
    }
    
    /* Run monitoring loop */
    monitor_loop();
    
    /* Clean shutdown */
    nvml_cleanup();
    
    /* Clear screen on exit */
    safe_printf(DASHBOARD_REFRESH);
    safe_printf("Monitoring stopped.\n");
    
    return EXIT_SUCCESS;
}
