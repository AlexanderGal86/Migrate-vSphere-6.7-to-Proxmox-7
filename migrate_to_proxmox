#!/bin/bash
#Для Alt Виртуализация 11 (Proxmox 7)
#Остановка при любой ошибке
#set -euo pipefail

# === Настройки ===
VCENTER="********"
VCENTER_USER="root"
VCENTER_PASS="**********" 
VM_NAME="REDOS-eDocGate"
PROXMOX_VMID=100
PROXMOX_STORAGE="local"
EXPORT_DIR="/opt/VipnetControl2012"

LOGFILE="$EXPORT_DIR/error.log"

# === Подготовка ===
mkdir -p "$EXPORT_DIR"
exec 2>>"$LOGFILE"   # stderr -> error.log

# Функция для выполнения команды с проверкой и сообщением
run_step() {
    local step_desc="$1"
    shift
    echo "[+] $step_desc"
    "$@" || { echo "? Ошибка на шаге: $step_desc" >&2; exit 1; }
}

# === Шаг 1: чистим старые файлы ===
run_step "Шаг 1: Чистим старые файлы в $EXPORT_DIR/$VM_NAME" \
   rm -f "$EXPORT_DIR"/"$VM_NAME"/*.ovf "$EXPORT_DIR"/"$VM_NAME"/*.mf "$EXPORT_DIR"/"$VM_NAME"/*.vmdk "$EXPORT_DIR"/"$VM_NAME"/*.nvram

# === Шаг 2: останавливаем VM на Proxmox ===
run_step "Шаг 2: Останавливаем VM $PROXMOX_VMID на Proxmox" \
  qm stop "$PROXMOX_VMID"

# === Шаг 3: экспорт через ovftool ===
run_step "Шаг 3: Экспортируем VM $VM_NAME из vSphere в $EXPORT_DIR/$VM_NAME" \
   ovftool --noSSLVerify --powerOffSource --parallelThreads=8 \
       "vi://${VCENTER_USER}:${VCENTER_PASS}@${VCENTER}/${VM_NAME}" \
       "$EXPORT_DIR"

# === Шаг 4: удаляем старые диски ===
echo "[!] Шаг 4: Удаляем все диски VM"
OLDDISK_ID=""
OLDDISK_ID=$(qm config "$PROXMOX_VMID" | awk -F: '{print $1}' | grep -E '^(sata|scsi|virtio|ide)[0-9]*$')

# Проверка на пустоту переменной OLDDISK_ID
if [[ -n "$OLDDISK_ID" ]]; then
  echo "Удаляем $OLDDISK_ID"
  #run_step "Run: xargs -n1 qm disk unlink "$PROXMOX_VMID" $OLDDISK_ID" \
  echo "$OLDDISK_ID" | xargs -I {} qm disk unlink "$PROXMOX_VMID" --idlist {}  || true
else
  echo "Нет дисков для удаления." 
fi

# === Шаг 4.1: определяем UNLINKDISK_ID после удаления дисков ===
echo "Шаг 4.1: определяем 'UNLINKDISK_ID' после удаления дисков"
UNLINKDISK_ID=""
UNLINKDISK_ID=$(qm config "$PROXMOX_VMID" | awk -F: '{print $1}' | grep -E '^(unused)[0-9]*$')

# Проверка на пустоту переменной UNLINKDISK_ID
if [[ -n "$UNLINKDISK_ID" ]]; then
  echo "Удаляем файл с хранилища, если он есть"
  #run_step "Run: xargs -n1 qm set "$PROXMOX_VMID" -delete $UNLINKDISK_ID" \ 
  echo "$UNLINKDISK_ID" | xargs -I {} qm set "$PROXMOX_VMID" -delete {}  || true
else
  echo "Нет файлов для удаления."
fi

# === Шаг 5: ищем vmdk ===
VMDK_FILE=$(ls "$EXPORT_DIR"/"$VM_NAME"/*.vmdk | head -n1)
if [[ -z "$VMDK_FILE" ]]; then
  echo "? Ошибка: vmdk не найден в $EXPORT_DIR" >&2
  exit 1
fi
echo "[+] Шаг 5: Найден диск: $VMDK_FILE"

# === Шаг 6: импортируем диск в Proxmox ===
run_step "Шаг 6: Импортируем диск в Proxmox" \
    qm importdisk "$PROXMOX_VMID" "$VMDK_FILE" "$PROXMOX_STORAGE" --format qcow2

NEWDISK_ID=""
NEWDISK_ID=$(pvesh get /nodes/$(hostname)/qemu/$PROXMOX_VMID/config | awk '/(sata|scsi|virtio|ide|unused)[0-9]+/ {print $4}' | tail -n1)

echo "[+] Шаг 6: Новый диск: $NEWDISK_ID"

# === Шаг 7: подключаем новый диск как sata0 ===
echo "[+] Шаг 7:"
run_step "Подключение нового диска как sata0" \
    qm set "$PROXMOX_VMID" --sata0 "$NEWDISK_ID"
    
# === Шаг 8: настройка загрузки
echo "[+] Шаг 8:"
run_step "Настройка порядка загрузки - sata0" \
    qm set $PROXMOX_VMID --boot order=sata0

# === Шаг 9: запускаем VM на Proxmox ===
run_step "Запуск VM $PROXMOX_VMID" \
    qm start "$PROXMOX_VMID"

echo "[?] Миграция завершена"
echo "Ошибки (если были) смотри в $LOGFILE"
