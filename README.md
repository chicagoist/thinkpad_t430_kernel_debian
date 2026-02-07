https://www.kernel.org/pub/linux/kernel/v6.x/linux-6.1.159.tar.gz

---

### Подготовка (Чистый старт)

Сначала вернем базу в исходное стабильное состояние:

```bash

cd ~/kernel-build
rm linux-image-6.1.159-t430-custom-t430-optimized*.deb
rm linux-headers-6.1.159-t430-custom-t430-optimized*.deb
# Или просто удалите все deb файлы в папке, чтобы было чисто:
rm *.deb

make mrproper
cp /boot/config-$(uname -r) .config
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
make olddefconfig
make menuconfig

```

Теперь, находясь в меню, следуйте по пунктам.

---

### 1. Процессор (Optimization for Ivy Bridge)

Ваш i5-3320M — это архитектура Ivy Bridge. Стандартное ядро Debian собрано для "Generic x86", чтобы работать везде. Мы это исправим.

* **Processor type and features**
* `Processor family`: Смените с **Generic-x86-64** на **Core 2/newer Xeon**.
* *Почему:* Это активирует оптимизации инструкций именно под Intel, убирая код для старых AMD K8/Athlon.


* `Supported processor vendors`:
* Зайдите внутрь и **отключите** `Support AMD processors`.
* **Отключите** `Support Hygon processors`.
* **Отключите** `Support Centaur processors`.
* *Оставьте* `Support Intel processors`.


* `Microcode loading`:
* Зайдите внутрь.
* **Отключите** `AMD microcode loading`.
* *Оставьте* `Intel microcode loading`.





### 2. Видеокарты (Удаляем AMD/Radeon)

У вас Nvidia NVS 5400M + Intel HD 4000. Драйверы AMD Radeon занимают огромное место.

* **Device Drivers** -> **Graphics support**
* **Отключите** `AMD GPU` (amdgpu).
* **Отключите** `ATI Radeon`.
* *Важно:* **Оставьте** `Intel 8xx/9xx... (i915)`.
* *Важно:* **Оставьте** `Nouveau (NVIDIA)`. Даже если вы будете ставить проприетарный драйвер 390.xx, модуль Nouveau нужен ядру для корректного запуска и последующей блокировки (blacklist). Без него могут быть проблемы с переключением графики.



### 3. Сеть (Wi-Fi и Ethernet)

У вас Intel Centrino 6205 (Wi-Fi), Ericsson (3G) и Intel 82579LM (LAN). Удаляем конкурентов.

* **Device Drivers** -> **Network device support** -> **Wireless LAN**
* **Отключите** `Broadcom` (все пункты).
* **Отключите** `Realtek` (все пункты).
* **Отключите** `Atheros/Qualcomm` (все пункты).
* **Отключите** `Mediatek`.
* **Отключите** `Marvell`.
* *Проверьте:* Убедитесь, что внутри **Intel devices** включен `Intel Wireless WiFi Next Gen AGN - Wireless-N/Advanced-N/Ultimate-N (iwlwifi)`.


* **Device Drivers** -> **Network device support** -> **Ethernet driver support**
* **Отключите** `Broadcom`.
* **Отключите** `Realtek`.
* **Отключите** `Marvell`.
* *Проверьте:* Внутри **Intel devices** должен быть включен `Intel(R) PRO/1000 Gigabit Ethernet support` (модуль `e1000e`). Это ваша сетевая карта.



### 4. Звук

У вас Intel HD Audio.

* **Device Drivers** -> **Sound card support** -> **Advanced Linux Sound Architecture** -> **PCI sound devices**
* Здесь можно отключить всё (Creative, C-Media, ASUS Xonar...), **КРОМЕ** `Intel HD Audio`.



### 5. Виртуализация (Hyper-V Guest)

Вы просили поддержку работы ВНУТРИ виртуальной машины (как гость). Удаляя "лишнее", нельзя задеть эти драйверы.

* **Device Drivers** -> **Microsoft Hyper-V guest support**
* Убедитесь, что стоит `<Y>` (звездочка) или `<M>`.


* **Device Drivers** -> **Input device support** -> **Hardware I/O ports** -> `Microsoft Hyper-V Mouse` `<M>`.
* **Device Drivers** -> **Graphics support** -> `Microsoft Hyper-V Synthetic Video support` `<M>`.

### 6. Энергоэффективность (AC vs DC)

Для T430 и ядер 6.x лучшая практика — использовать драйвер `intel_pstate`.

* **Power management and ACPI options** -> **CPU Frequency scaling**
* `Default scaling governor`: Выберите **performance**.
* *Почему:* `intel_pstate` в режиме performance работает умнее, чем старые драйверы. Он умеет сбрасывать частоту до минимума в простое. А всю тонкую настройку (переключение профилей при отключении кабеля) сделает утилита TLP в userspace, а не ядро.


* Убедитесь, что `Intel P state control` включен `<Y>`.



### 7. Специфика ThinkPad T430

* **Device Drivers** -> **X86 Platform Specific Device Drivers**
* Проверьте `ThinkPad ACPI Laptop Extras`. Должно быть `<M>` или `<Y>`.
* Внутри включите `Console audio control mechanism` (для кнопок громкости).



### 8. Отключение Отладки (Критично для размера!)

Это то, что делает ядро "тяжелым", а не драйверы AMD.

* **Kernel hacking** -> **Compile-time checks and compiler options** -> **Debug information**
* Выберите **Disable debug information**.
* *Результат:* Размер модулей уменьшится в 10 раз.



---

### Итоговая сборка

После внесения этих изменений:

1. Нажмите `Save` (сохранить в `.config`).
2. Выйдите из меню (`Exit`).
3. Запустите сборку:

```bash
make -j4 LOCALVERSION=-t430-optimized bindeb-pkg

```

### Что вы получите?

* **Минус ~3000 модулей** от AMD, Broadcom, Radeon и т.д.
* **Размер пакета:** около 40-60 МБ (вместо стандартных 500+ МБ с debug info).
* **Оптимизация:** Инструкции процессора теперь `march=core2` (Ivy Bridge совместимый), а не общий `generic`.
* **Безопасность:** Мы не трогали USB, файловые системы и контроллеры дисков, поэтому риск получить "кирпич" минимален.
