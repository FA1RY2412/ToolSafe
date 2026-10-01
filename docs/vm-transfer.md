# Передача ToolSafe в Ubuntu VM

Основна копія знаходиться на Mac у `Lab01/Work/ToolSafe`. Архів `../ToolSafe-vm.tar.gz` і його `../ToolSafe-vm.tar.gz.sha256` лежать поза репозиторієм. Архів містить джерела та документацію, без `.git` і `firmware/build`. Він не призначений для показу Git-історії; історію показувати з основної копії.

Правила: [Study/VM-transfer.md](../../../../../../../../VM-transfer.md). Наведені команди передачі та VM-перевірки ще не запускались. Адресу `ubuntu-vm` та користувача треба звірити з реальною VM; цей приклад не підтверджує доступ по SSH. Не редагувати одну роботу одночасно на Mac і у VM.

## Mac до VM

Запустити Ubuntu 24.04 VM перед передачею. Для SSH підставити перевірену адресу. Перший крок лише перегляд:

```sh
ssh pliffdax@ubuntu-vm 'mkdir -p /home/pliffdax/study/2026-autumn/SP/ToolSafe/Lab01/incoming'
rsync -avn '/Users/pliffdax/Projects/university/Study/People/Голуб Денис Дмитрович/2026-Fall/SP/Lab01/Work/ToolSafe-vm.tar.gz' '/Users/pliffdax/Projects/university/Study/People/Голуб Денис Дмитрович/2026-Fall/SP/Lab01/Work/ToolSafe-vm.tar.gz.sha256' 'pliffdax@ubuntu-vm:/home/pliffdax/study/2026-autumn/SP/ToolSafe/Lab01/incoming/'
```

Після порівняння списку повторити rsync з `-av`. Не додавати `--delete`. Альтернатива без SSH: скопіювати обидва файли через shared folder VM. Усередині Ubuntu:

```sh
cd /home/pliffdax/study/2026-autumn/SP/ToolSafe/Lab01/incoming
sha256sum -c ToolSafe-vm.tar.gz.sha256
tar -tzf ToolSafe-vm.tar.gz
```

Перед розпакуванням перевірити, що `../Work/ToolSafe` не існує. Якщо існує, порівняти його й використати окрему incoming-копію; не перезаписувати. Для нової копії:

```sh
mkdir -p ../Work
test ! -e ../Work/ToolSafe && tar -xzf ToolSafe-vm.tar.gz -C ../Work
cd ../Work/ToolSafe
mkdir -p ../vm-evidence
cat /etc/os-release | tee ../vm-evidence/os-release.txt
uname -m | tee ../vm-evidence/architecture.txt
docker version | tee ../vm-evidence/docker-version.txt
docker info > ../vm-evidence/docker-info.txt
set -o pipefail
make clean && make build IMAGE=toolsafe-stm32-build:local 2>&1 | tee ../vm-evidence/build-1.log
sha256sum firmware/build/firmware.elf firmware/build/firmware.hex firmware/build/firmware.bin > ../vm-evidence/build-1.sha256
make clean && make build IMAGE=toolsafe-stm32-build:local 2>&1 | tee ../vm-evidence/build-2.log
sha256sum firmware/build/firmware.elf firmware/build/firmware.hex firmware/build/firmware.bin > ../vm-evidence/build-2.sha256
diff -u ../vm-evidence/build-1.sha256 ../vm-evidence/build-2.sha256
```

Виконувати наступний крок тільки після коду завершення 0 попереднього. Відсутність diff означає побайтну рівність двох збірок на цій VM. SHA хоста й VM порівнювати окремо: різна архітектура образа або версія компілятора може змінити ELF. Не приписувати хостові логи запуску в Ubuntu.

## Wokwi та повернення результатів

VS Code з Wokwi вже знайдено на Mac. У VM наявність VS Code/ліцензії не перевірена. Можна збирати в Ubuntu і симулювати на Mac: передати саме ELF/HEX з перевіреної VM-копії, спочатку в окрему incoming-папку, звірити SHA256 і лише потім свідомо замінити локальні згенеровані артефакти. `wokwi.toml` посилається на `firmware/build/firmware.hex` та `.elf`. Для цього запуску зберегти нові знімки з ready/tick/LED і зазначити походження бінарника. Trial не активувати.

Повернення логів з VM спочатку в `../VM-Inbox/`, з rsync `-avn`, потім `-av` після порівняння. Не копіювати `.git`, не змінювати основні файли навмання. У звіт додавати лише перевірені результати та справжні знімки.
