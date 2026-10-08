[graidutil for Ubuntu檔案下載](https://download.graidtech.com/misc/tools/graid_log_collector/linux/graidutil-1.0.0-30-x86_64.deb)
[Python Script檔案下載](https://download.graidtech.com/misc/tools/graid_log_collector/linux/offline-archive-tool.zip)

**Install graidutil**
```
sudo dpkg -i <graidutil-package.deb>
graidutil --version
graidutil --help
```

**Collect profile**
```
graidutil collect data -o /tmp/installed_versions.txt
```

**Check kernel**
```
python check_kernel.py --installed installed_versions.txt
```

**Generate offline packages**
```
python3 offline.py --installed installed_versions.txt

# output_tar/<distro><major>_offline_packages.tar.gz
```

**Install offline packages**
```
sudo graidutil install ubuntu24_offline_packages.tar.gz
```

**Collect Log**
```
sudo graidutil collect log -o /var/tmp/graid-case-001
```