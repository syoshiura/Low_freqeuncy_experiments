# EMBERS

高精度な衛星 TLE データのダウンロードと軌道プロットツール。  
https://www.orbcomm.co.jp/info/satellite.html
https://embers.readthedocs.io/en/latest/embersbyexample.html

Mac+condaで環境構築がうまくいかなかったので、Singularityを使う。

embers用のsandbox作成 
```SH
sudo singularity build --sandbox embers-sandbox/ Singularity.def 
```

Singularity.defの中身
```yml
Bootstrap: docker
From: ubuntu:20.04

%labels
    Author Yoshiura Shintaro
    Version 1.0

%environment
    export PATH=/opt/embers-env/bin:$PATH
    export LC_ALL=C.UTF-8
    export LANG=C.UTF-8

%post
    export DEBIAN_FRONTEND=noninteractive

    # 基本パッケージと tzdata を同時にインストール
    apt-get update && apt-get install -y \
        python3.8 python3.8-venv python3-pip \
        build-essential git wget curl ca-certificates \
        libffi-dev libssl-dev python3.8-dev tzdata \
        && rm -rf /var/lib/apt/lists/*

    # タイムゾーンを非対話で UTC に設定
    ln -fs /usr/share/zoneinfo/Etc/UTC /etc/localtime
    dpkg-reconfigure --frontend noninteractive tzdata

    # 仮想環境作成
    python3.8 -m venv /opt/embers-env
    . /opt/embers-env/bin/activate

    # pip, setuptools, wheel のアップグレード
    pip install --upgrade pip setuptools wheel

    # 必要な Python パッケージのインストール
    pip install "six==1.15.0" "spacetrack==0.13.6" "sgp4>=2.12" "skyfield==1.22"
    pip install embers

%runscript
    echo "Singularity container for embers"
    exec /bin/bash

%test
    source /opt/embers-env/bin/activate
    python -c "import embers; print('Embers imported successfully')"

```

SandboxからSifに変換したので誰でも使えるはずだが、570MBくらいある。
```SH
sudo singularity build embers.sif embers-sandbox/
```

---

# 使い方

EMBERSのShellに入る。作業ディレクトリ(ここでは"/home/work/EMBERS")をバインドしておく。
```SH
singularity shell --bind /home/work/EMBERS:/EMBERS embers.sif
cd /EMBERS/
```

作業ディレクトリにはscript_download.pyやplot_path.pyを保存しておく。

## ダウンロード
TLE データを [Space-Track.org](https://www.space-track.org/) から取得する。

```bash
python3 script_download.py
```

script_download.pyの中身例:
``` py
from embers.sat_utils.sat_list import download_tle, norad_ids


start_date = "2019-10-01"
stop_date = "2019-10-10"
out_dir = "./embers_out/sat_utils/TLE"
n_ids = norad_ids()

# Make account on space-track.org and enter credentials below
st_ident = "[mail]"
st_pass = "[passward]"

download_tle(
    start_date,
    stop_date,
    n_ids,
    st_ident=st_ident,
    st_pass=st_pass,
    out_dir=out_dir,
)

print(f"TLE files saved to {out_dir}")
```

**実行例**
```text
Starting TLE download
Grab a coffee, this may take a while
downloading tle for ORBCOMM-X satellite [21576] from space-tracks.org
downloading tle for ORBCOMM FM 1 satellite [23545] from space-tracks.org
downloading tle for ORBCOMM FM 2 satellite [23546] from space-tracks.org
downloading tle for ORBCOMM FM 8 satellite [25112] from space-tracks.org
downloading tle for ORBCOMM FM 10 satellite [25113] from space-tracks.org
downloading tle for ORBCOMM FM 11 satellite [25114] from space-tracks.org
downloading tle for ORBCOMM FM 12 satellite [25115] from space-tracks.org
downloading tle for ORBCOMM FM 9 satellite [25116] from space-tracks.org
```


## 軌道をプロット
```SH
python3 plot_path.py
```

plot_path.pyの中身例：
```py 
from pathlib import Path
from embers.sat_utils.sat_ephemeris import save_ephem

sat="25417"
tle_dir="./embers_out/sat_utils/TLE"
cadence = 4
location = (-26.703319, 116.670815, 337.83)
out_dir = "./"
alpha = 0.5
status = save_ephem(sat, tle_dir, cadence, location, alpha, out_dir)
print(status)
```

## 軌道情報をnpzに出力。
```SH
python3 time_alt_az.py
```

time_alt_az.pyの中身例：
```py 
import numpy as np
from embers.sat_utils.sat_ephemeris import load_tle, epoch_ranges, epoch_time_array, sat_pass, ephem_data
sat="25417"
tle_dir="./embers_out/sat_utils/TLE"
sats, epochs = load_tle(tle_dir + '/'+ sat + '.txt')
epoch_range = epoch_ranges(epochs)
index_epoch = 0     # select first time interval from epoch_range
cadence = 10        # evaluate satellite position every 10 seconds
t_arr, index_epoch = epoch_time_array(epoch_range, index_epoch, cadence)
MWA = (-26.703319, 116.670815, 337.83)   # gps coordinates of MWA Telescope
passes, alt, az = sat_pass(sats, t_arr, index_epoch, location=MWA)

time_array, sat_alt, sat_az = ephem_data(t_arr, passes[0], alt, az)
np.savez("sat_time_alt_az.npz", time=time_array, alt=sat_alt, az=sat_az)
```

