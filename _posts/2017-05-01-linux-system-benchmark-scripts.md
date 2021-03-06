---
layout: post
title:  "Linux Server Benchmarking Scripts"
date:   2017-05-01
categories: servers
tags: [benchmarks, servers, performance]
image:
  feature: abstract-1.jpg
  credit:
---

> Goal is a simple set of bash scripts which allow rapid assessment of any given hardware.

Currently only CPU & HDD tests are wired up. Other tests are a work-in-progress, added as needed.

#### CREATE BENCHMARK SHORTCUT ALIASES

```sh
# COPY + PASTE THE FOLLOWING TO CREATE FOLDER & MAIN SCRIPT(S)
# Create folder for results & scripts
export BENCH_DIR=$HOME/benchmarks
mkdir -p $BENCH_DIR/results

cat << 'EOT' >> $HOME/benchmarks/bench-library.sh
#!/bin/bash
set -e

# Install some deps
if [ "$(which sysbench)" == "" -o "$(which inxi)" == "" -o "$(which tcpdump)" == "" ]; then
  apt-get update && apt-get install -y sysbench inxi htop iotop tcpdump hddtemp
fi
# Variables
export DATE_TAG=`date +%F` #YYYY-MM-DD
export CPU_CORES="$([ -e /proc/cpuinfo ] && grep -sc ^processor /proc/cpuinfo || sysctl -n hw.ncpu)"
export BENCH_DIR=$HOME/benchmarks/

mkdir -p $BENCH_DIR

function benchCpu() {
  thread_limit=${1:$CPU_CORES}
  prime_limit=${2:-20000}

  if [ $CPU_CORES -lt `expr 1 + $thread_limit` ]; then
    printf "\n\n${yellow}ALERT: Skipping tests limited by \"${thread_limit} thread test\"\n${cyan}Not enough CPU Cores ($CPU_CORES)  ${reset}\n\n"
  else
    printf "\n\n${yellow}ALERT: Skipping tests limited by \"${thread_limit} thread test\"\n${reset}"

    sysbench --test=cpu \
      --cpu-max-prime=${prime_limit} \
      --num-threads=${CPU_CORES} \
      run | tee -a $BENCH_DIR/results/cpu-test.log
  fi
}

# benchSingleDisk seqrd 120G 8K 300
function benchSingleDisk () {
  sysbench --test=fileio --init-rng=on  --file-test-mode=${1:-seqrd} --file-block-size=${3:-64K} \
    --num-threads=${CPU_CORES} --max-time=${4:-180} --file-total-size=${2:-60G} \
    --max-requests=0 run | tee -a $BENCH_DIR/results/sysbench-fileio.log
}


# benchDisk - tests random read & write, and sequential r, and sequential write, before final cleanup.
function benchDisk() {
  #   Generates test files - up to 75% of your free space - in local dir, then runs the 3 tests (up to 20 minutes each)
  freeSpace=`df -k . | tail -1 | awk '{print $4}'`
  freeSpace="${freeSpace//G|T/}"
  testSize=$(awk "BEGIN {print ($freeSpace / 1024 / 1024) * 0.75; exit}")
  testSize=${testSize}G
  printf "####>>> \nWriting $testSize test data to ${PWD}...\n"

  benchSingleDisk seqrd ${testSize} 8K 300
  benchSingleDisk seqwr ${testSize} 8K 300
  benchSingleDisk seqrw ${testSize} 8K 300
  benchSingleDisk rndrd ${testSize} 8K 300
  benchSingleDisk rndwr ${testSize} 8K 300
  benchSingleDisk rndrw ${testSize} 8K 300

  benchSingleDisk seqrd ${testSize} 64K 300
  benchSingleDisk seqwr ${testSize} 64K 300
  benchSingleDisk seqrw ${testSize} 64K 300
  benchSingleDisk rndrd ${testSize} 64K 300
  benchSingleDisk rndwr ${testSize} 64K 300
  benchSingleDisk rndrw ${testSize} 64K 300

  printf "\n\n####>>> \nCOMPLETED TESTS! Great Success!!! \n\n\n"
}

EOT

chmod +x $BENCH_DIR/*.sh
source $HOME/benchmarks/bench-library.sh

```

#### Step 2: CREATE RUNNER SCRIPT

```sh
cat << 'EOT' >> tee $HOME/benchmarks/run-bench.sh
#!/bin/bash
set -e

source ./bench-library.sh

# Benchmark HDD Speed (in Current Directory)
###########
benchDisk

# Benchmark CPU - trying different thread counts (and work sizes)
# It'll automatically skip test if we don't have enough cores (to have an impact)
# NB: results comparable between different hardware - up to their same CPU CORE #.
###########
benchCpu 1
benchCpu 4
benchCpu 8  50000
benchCpu 12 100000
benchCpu 16 100000
benchCpu 32 250000
benchCpu 48 500000
benchCpu 64 2000000

EOT

chmod +x $BENCH_DIR/*.sh





```

## Usage Examples:

Make sure to `source ~/benchmarks/bench-library.sh` before running the following commands manually.

```sh
benchCpu 8   250000
benchCpu 16  250000
benchDisk
```


<!--# I/O - Live Monitor
1. System: iotop
1. Per command: dtrace/ltrace/strace
-->



