# ar9271-gki-buildername: Build AR9271 GKI 6.12.38 Modules

on:
  workflow_dispatch:

jobs:
  build-modules:
    runs-on: ubuntu-22.04
    steps:
      - name: Maximize Build Disk Space
        run: |
          sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc
          df -h

      - name: Install Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y git curl python3 build-essential bc bison flex libssl-dev libelf-dev zip rsync

      - name: Install Repo Tool
        run: |
          mkdir -p ~/bin
          curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
          chmod a+x ~/bin/repo
          echo "$HOME/bin" >> $GITHUB_PATH

      - name: Sync AOSP GKI android16-6.12 Source
        run: |
          mkdir -p ~/gki && cd ~/gki
          git config --global user.name "Builder"
          git config --global user.email "builder@example.com"
          repo init -u https://android.googlesource.com/kernel/manifest -b common-android16-6.12 --depth=1
          repo sync -c -j$(nproc) --no-tags

      - name: Checkout Exact Device Commit & Configure AR9271
        run: |
          cd ~/gki/common
          git fetch --depth=50 origin b575a0b6e647 || true
          git checkout b575a0b6e647 || echo "Using tip of common-android16-6.12"
          
          cat << 'EOF' >> arch/arm64/configs/gki_defconfig
          CONFIG_LOCALVERSION="-android16-5-gb575a0b6e647-ab14355190-4k"
          CONFIG_LOCALVERSION_AUTO=n
          CONFIG_CFG80211=m
          CONFIG_CFG80211_WEXT=y
          CONFIG_MAC80211=m
          CONFIG_MAC80211_MESH=y
          CONFIG_WLAN=y
          CONFIG_WLAN_VENDOR_ATH=y
          CONFIG_ATH_COMMON=m
          CONFIG_ATH9K_HW=m
          CONFIG_ATH9K_COMMON=m
          CONFIG_ATH9K_HTC=m
          EOF

      - name: Build Kernel Modules with Kleaf
        run: |
          cd ~/gki
          tools/bazel build --lto=thin //common:kernel_aarch64_dist

      - name: Download AR9271 Firmware & Package Magisk Module
        run: |
          mkdir -p ~/magisk_mod/META-INF/com/google/android
          mkdir -p ~/magisk_mod/modules
          mkdir -p ~/magisk_mod/system/vendor/firmware/ath9k_htc

          # Copy compiled .ko modules
          DIST=~/gki/out/android16-6.12/dist
          for mod in rfkill.ko cfg80211.ko mac80211.ko ath.ko ath9k_hw.ko ath9k_common.ko ath9k_htc.ko; do
            [ -f "$DIST/$mod" ] && cp "$DIST/$mod" ~/magisk_mod/modules/
          done

          # Fetch official AR9271 firmware
          curl -L https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/ath9k_htc/htc_9271-1.4.0.fw \
            -o ~/magisk_mod/system/vendor/firmware/htc_9271.fw
          cp ~/magisk_mod/system/vendor/firmware/htc_9271.fw ~/magisk_mod/system/vendor/firmware/ath9k_htc/htc_9271-1.4.0.fw

          # Create Magisk installer scripts
          curl -L https://raw.githubusercontent.com/topjohnwu/Magisk/master/scripts/module_installer.sh \
            -o ~/magisk_mod/META-INF/com/google/android/update-binary
          echo "#MAGISK" > ~/magisk_mod/META-INF/com/google/android/updater-script

          cat << 'EOF' > ~/magisk_mod/module.prop
          id=nethunter_ar9271
          name=NetHunter Atheros AR9271 (GKI 6.12.38)
          version=1.0
          versionCode=1
          author=Custom
          description=GKI 6.12.38 cfg80211, mac80211, ath9k_htc modules and firmware for TECNO CL6k
          EOF

          cat << 'EOF' > ~/magisk_mod/service.sh
          #!/system/bin/sh
          MODDIR=${0%/*}
          echo -n "$MODDIR/system/vendor/firmware" > /sys/module/firmware_class/parameters/path
          insmod $MODDIR/modules/rfkill.ko 2>/dev/null
          insmod $MODDIR/modules/cfg80211.ko
          insmod $MODDIR/modules/mac80211.ko
          insmod $MODDIR/modules/ath.ko
          insmod $MODDIR/modules/ath9k_hw.ko
          insmod $MODDIR/modules/ath9k_common.ko
          insmod $MODDIR/modules/ath9k_htc.ko
          EOF
          chmod +x ~/magisk_mod/service.sh

          cd ~/magisk_mod
          zip -r ~/nethunter_ar9271_6.12.38.zip .

      - name: Upload Flashable Magisk Module
        uses: actions/upload-artifact@v4
        with:
          name: nethunter_ar9271_6.12.38
          path: ~/nethunter_ar9271_6.12.38.zip
