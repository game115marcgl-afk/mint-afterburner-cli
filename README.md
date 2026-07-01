# Mint Afterburner CLI 🚀

Lekki, otwartoźródłowy (FOSS) odpowiednik programów typu MSI Afterburner / RivaTuner (RTSS) w wersji dla terminala. Stworzony z myślą o minimalnym zużyciu zasobów, bez telemetrycznych i ciężkich interfejsów graficznych.

Projekt komunikuje się bezpośrednio ze sterownikiem graficznym NVIDIA poprzez natywne API biblioteki **NVML (NVIDIA Management Library)**. Idealny dla starszych i nowszych architektur (w tym serii Pascal GTX 1050/1060).

## 📊 Funkcje programu
- Monitoring zużycia rdzenia GPU na żywo (%)
- Monitoring zużycia pamięci VRAM (używana / całkowita w MiB)
- Odczyt temperatur w czasie rzeczywistym (°C)
- Monitorowanie prędkości obrotowej wentylatorów (%)
- Wyświetlanie aktualnych taktowań zegara rdzenia oraz pamięci (MHz)
- Bezpieczny kod pozbawiony podatności formatowania tekstu (`snprintf`, `vprintf`)

## 🛠️ Wymagania systemowe
Do poprawnego działania i kompilacji wymagane są oficjalne nagłówki deweloperskie NVIDIA:
```bash
sudo apt update && sudo apt install nvidia-cuda-toolkit

Kompilacja i uruchomienie

Sklonuj repozytorium i wykonaj automatyczną kompilację przy użyciu pliku Makefile:

Bash
cd mint-afterburner-cli
make
./mint-afterburner-cli
