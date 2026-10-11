# 📂 Yandex Disk Folder Downloader

Консольная утилита для скачивания больших папок с Яндекс.Диска. Обходит ошибки `500` и таймауты сервера, сохраняя полную иерархию папок и файлов.

---

## 🛠️ Что нужно для работы

Убедись, что у тебя установлен Python, и поставь единственную зависимость:
```bash
pip install requests
```

---

## 🐍 Полный код скрипта (`download_files.py`)

Скопируй этот код в файл `download_files.py`:

```python
import os
import sys
import time
import argparse
import requests
from urllib.parse import urlencode, urlparse

HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
}

session = requests.Session()
session.headers.update(HEADERS)

def is_yandex_disk_url(url: str) -> bool:
    try:
        parsed = urlparse(url)
        valid_domains = ('disk.yandex.ru', 'disk.yandex.com', 'yadi.sk')
        netloc = parsed.netloc.lower()
        return any(netloc == domain or netloc.endswith('.' + domain) for domain in valid_domains)
    except Exception:
        return False

def process_and_download_folder(public_key: str, path: str = "", current_rel_path: str = "", output_dir: str = "arizona_files"):
    base_url = 'https://cloud-api.yandex.net/v1/disk/public/resources?'
    params = {'public_key': public_key, 'limit': 1000}
    if path:
        params['path'] = path

    url = base_url + urlencode(params)
    print(f"📂 Сканирую и качусь по папке: {path if path else 'Корень'}...")

    data = None
    for attempt in range(1, 4):
        try:
            time.sleep(0.3)
            response = session.get(url, timeout=35)
            response.raise_for_status()
            data = response.json()
            break
        except (requests.exceptions.Timeout, requests.exceptions.ConnectionError):
            print(f"⚠️ Таймаут на папке '{path}' (попытка {attempt}/3), пробуем снова...")
            time.sleep(2)
        except Exception as e:
            print(f"⚠️ Ошибка при чтении папки '{path}': {e}")
            return

    if not data:
        print(f"❌ Не удалось получить содержимое папки '{path}'")
        return

    embedded = data.get('_embedded')
    if not embedded:
        return

    items = embedded.get('items', [])

    for item in items:
        if item['type'] == 'file':
            filename = item['name']
            file_url = item['file']
            rel_path = os.path.join(current_rel_path, filename)
            file_path = os.path.join(output_dir, rel_path)

            os.makedirs(os.path.dirname(file_path), exist_ok=True)

            print(f"📥 Скачиваю: {rel_path}...")

            for attempt in range(1, 3):
                try:
                    with session.get(file_url, stream=True, timeout=40) as r:
                        r.raise_for_status()
                        with open(file_path, 'wb') as f:
                            for chunk in r.iter_content(chunk_size=8192):
                                if chunk:
                                    f.write(chunk)
                    break
                except Exception as e:
                    if attempt == 2:
                        print(f"❌ Ошибка при скачивании файла {rel_path} после 2 попыток: {e}")
                    else:
                        print(f"⚠️ Сбой на файле {rel_path}, повтор ({attempt}/2)...")
                        time.sleep(2)

        elif item['type'] == 'dir':
            sub_path = item.get('path')
            sub_rel_path = os.path.join(current_rel_path, item['name'])
            process_and_download_folder(public_key, sub_path, sub_rel_path, output_dir)

def main():
    parser = argparse.ArgumentParser(description="Загрузчик файлов с Яндекс Диска с сохранением иерархии папок на лету.")
    parser.add_argument('-u', '--url', required=True, help="Публичная ссылка на папку Яндекс Диска")
    parser.add_argument('-o', '--output', default="arizona_files", help="Целевая папка для сохранения (по умолчанию: arizona_files)")

    args = parser.parse_args()

    if not is_yandex_disk_url(args.url):
        print(f"❌ Ошибка: '{args.url}' не является корректной ссылкой на Яндекс Диск.")
        print("💡 Ссылка должна начинаться с https://disk.yandex.ru/... или https://yadi.sk/...")
        sys.exit(1)

    print("🚀 Запуск скачивания на лету...")
    os.makedirs(args.output, exist_ok=True)

    process_and_download_folder(args.url, output_dir=args.output)

    print(f"\n✅ Готово! Все доступные файлы сохранены в: {os.path.abspath(args.output)}")

if __name__ == "__main__":
    main()
```

---

## 🚀 Как использовать

### 1. Запуск через терминал (CLI)
Передай ссылку на папку через флаг `-u` и (опционально) путь для сохранения через `-o`:

```bash
python download_files.py --url "https://disk.yandex.ru/d/p3T24tLfZBCosA" --output "arizona_files"
```

**Доступные аргументы:**
* `-u`, `--url` — **Обязательно.** Публичная ссылка на папку Яндекс.Диска (`disk.yandex.ru` или `yadi.sk`).
* `-o`, `--output` — *Необязательно.* Папка, куда будут скачаны файлы (по умолчанию создастся папка `arizona_files` в текущей директории).

---

### 2. Запуск в один клик из VS Code (по F5)

Если не хочется постоянно писать команды вручную, добавь этот конфиг в свой файл `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Download Yandex Folder",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "args": [
        "--url", "https://disk.yandex.ru/d/p3T24tLfZBCosA",
        "--output", "arizona_files"
      ]
    }
  ]
}
```
После этого достаточно открыть `download_files.py` в редакторе и нажать **`F5`** — скрипт автоматически запустится во встроенном терминале с нужными параметрами.
