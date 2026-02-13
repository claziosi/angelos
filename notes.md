# Créer un dossier temporaire
mkdir -p /tmp/cifs-package

# Copier les binaires
cp /sbin/mount.cifs /tmp/cifs-package/
cp /usr/bin/cifscreds /tmp/cifs-package/ 2>/dev/null || true

# Identifier les librairies nécessaires
ldd /sbin/mount.cifs

# Copier les librairies dépendantes (ajustez selon votre ldd)
mkdir -p /tmp/cifs-package/lib
cp /lib/x86_64-linux-gnu/libcap.so.* /tmp/cifs-package/lib/ 2>/dev/null || true
cp /lib/x86_64-linux-gnu/libkeyutils.so.* /tmp/cifs-package/lib/ 2>/dev/null || true
cp /lib/x86_64-linux-gnu/libtalloc.so.* /tmp/cifs-package/lib/ 2>/dev/null || true
cp /lib/x86_64-linux-gnu/libwbclient.so.* /tmp/cifs-package/lib/ 2>/dev/null || true

# Créer une archive
cd /tmp
tar -czf cifs-utils.tar.gz cifs-package/
