## Perf-Analysis Tool

### Why this tool:
   This `Perf-Analysis tool` will identify the potential root cause of the performance issue in the production edge and generate the report. This will help the developers and quality engineers to identify the performance issue beforehand. This `Perf-Analysis tool` makes use of `perf` tool which is a linux profiler tool.
   
### Input:

The input to this `Perf-Analysis tool` is a json file that contains the following,
   
1. `Total Duration` - Total test duration
2. `Sampling frequency interval` - Interval at which the samples have to be taken. For example, if `Total Duration` is 10 seconds and `Sampling Frequency Interval` is 2 seconds, then 6 samples would be collected at the frequency of 2 seconds interval (ie, at 0s, 2s, 4s, 6s, 8s, 10s respectively)
3. `Output directory` - Specifies where to locate the resultant diag dump. Default location is current working directory.
4. List of processes, commands and counters to monitor along with optional trigger value. This tool have intelligence to analyse the handoff queue drops whereas `top -H -n 1` and `cat /opt/vc/lib/python/hardwareinfo.py` are custom user commands. 
For example,

        {
        	"duration" : 15,
        	"sample_frequency" : 5,
        	"monitor" : {
        		"process" : [
        			{
        				"name" : "all"
        			},
        			{
        				"name" : "edged"
        			}
        		],
        		"commands" : [
        			{
        				"name" : "handoff"
        			},
        			{
        				"name" : "top -H -n 1"
        			},
        			{
        				"name" : "cat /opt/vc/lib/python/hardwareinfo.py"
        			}
        		],
        		"counters" : [
        			{
        				"name" : "dpdk_counters"
        			},
        			{
        				"name" : "memb.mod_vc_link_cos_qos_t.tot_bytes",
        				"trigger" : 11750
        			},
        			{
        				"name" : "memb.mod_vc_qos_calc_deqrate_slot_t.tot_bytes"
        			}		
        		]
        	},
        	"debug" : [...],
        	"diagnostics" : {...}
        }

5. List of what to filter - Items in these list were filtered to get the top x critical threads with maximum CPU utilization and drops.


        {
        	"duration" : 15,
        	"sample_frequency" : 5,
        	"monitor" : {
        		"cpu" : [...],
        		"commands" : [...],
        		"counters" : [...]
        	},
        	"debug" : [
        		"cpu",
        		"handoff_drops",
        		"counters"
        	],
        	"diagnostics" : {...}
        }

6. Diag tool(here perf) and its meta data - `perf` is a profiler tool for a linux based system. This make use of the following three perf commands:


    1. `Perf Record` - This runs a command and gathers a performance counter profile from it, into perf.data - without displaying anything. This file can then be inspected later on, using perf report.
        - `Sleep` - Sleep indicates the wait time.
        - `Frequency` - Profile at this frequency. Default is 4000 Hz.
        - `number_of_record` - Specify the number of profile to be done.
        - `delay_between_record` - Interval between each profile.

    2. `Perf Sched` - This record the scheduling events of an arbitrary workload into perf.data. This file can then be inspected later on, using `perf sched latency` to report the per task scheduling latencies and other scheduling properties of the workload.
        - `Sleep` - Sleep indicates the wait time.
        - `number_of_sched` - Specify the number of sched record to be done.
        - `delay_between_sched` - Interval between each sched.
        - `latency_key` - Key based on which the latency report to be sorted. `runtime, switch, avg, max` are some of the keys.

    3. `Perf Stat` - This runs a command and gathers performance counter statistics from it.
        - `Sleep` - Sleep indicates the wait time.
        - `Events` - Event names (use perf list to list all events)


                {
                    "duration" : 15,
                    "sample_frequency" : 5,
                    "monitor" : {...},
                    "debug" : [...],
                    "diagnostics" : {
                        "perf" : {
                            "record" : {
                                "sleep" : 3,
                                "frequency" : 999,
                                "number_of_record" : 2,
                                "delay_between_record" : 2
                            },
                            "sched" : {
                                "sleep" : 3,
                                "number_of_sched" : 2,
                                "delay_between_sched" : 2,
                                "latency_key" : [
                                    "Switches",
                                    "Average delay"
                                ]
                            },
                            "stat" : {
                                "sleep" : 0.01,
                                "events" : [
                                    "dTLB-loads",
                                    "dTLB-load-misses",
                                    "dTLB-stores", 
                                    ...
                                ]
                            }
                        }
                    }
                }

The following are the command line inputs for various modes,

1. Default mode

        python index.py
    Optional Sub-options:

    1. -s -> save profile. This option will save the profiled binary files for both `perf record` and `perf sched`.

2. Threshold detection mode
    
        python index.py -T
    Optional Sub-options:
    
    1. -o -> Output file for threshold dump
    2. -d -> Total test duration 

3. Auto mode:
   
        python index.py -A
    Optional Sub-options:

    1. -i -> input threshold dump file
    2. -m -> preference mode with choice of choosing between `value` and `bandwidth`. Default is `value`. This represents the key for trigger check.
    3. -w -> window size. This specifies the number of zip to be maintained at any time. For `window size` 10, first zip will expire when 11th zip arrives.
    4. -l -> consecutive threshold exceed limit. For example, this will wait for `n` trigger/ threshold hits before starting perf collect, for `consecutive threshold exceed limit = n `
    5. -s -> save profile. This option will save the profiled binary files for both `perf record` and `perf sched`.
              
4. Offline mode:
   
        python index.py -F <zip file>
    < zip file > is the previously monitored zip for which the report have to to be generated.

### How it works:

1. Monitor

    1. This tool will monitor the edge and collect,
        - CPU utilization percentage of all the active process and its threads using `psutil` library of python.
        - Counter drops using the proprietary command `getcntr -c <counter_name>`. For dpdk_counters, `debug.py --dpdk_ports_dump` will give the list of dpdk enabled ports.
        - Handoff queue drops using the proprietary command `debug.py --handoff`.
    2. In addition, this will also execute the inputted custom commands if any. For example, `top -H -n 1`, `cat /opt/vc/lib/python/hardwareinfo.py`

    3. These parsed data are dumped to a JSON file.
2. Debug

    1. Meanwhile, the top processor consuming threads, and the threads with maximum drops have been identified (from the parsed JSON).

3. Diagnostics

    1. Then, the device is profiled using the following `perf` commands, and the results of which will be stored in a binary file for later analysis
        - `perf record` for collecting the profililing data,
        - `perf sched record` for collecting per task scheduling latency.
    2. Now the following analysis is done for those top threads,
        - The profile data analysis using `perf report`, 
        - `perf sched latency` for summarizing scheduler latencies by task, including average and maximum delay, 
        - Performance counter statistics using `perf stat`.  
    3. The results of these analysis forms the potential root cause report.
    4. All these files will be zipped for future offline reference. 

#### Various modes:

1. Offline mode:

    - This mode accepts the monitored zip file as a command line argument and generate the report, thus allows offline processing. The inputted zip will have monitored data (CPU stats, counters and handoff queue drops) and the perf profiled data.
    - From the inputted file, the top processor consuming threads, and the threads with maximum drops have been identified.
    - Having the perf binary file, the perf analysis will be done and the potential root cause report will be generated.


2. Threshold detection mode:

    - This mode will monitor the CPU utilization, counters and handoff queue drops for the given duration, and maintain the list of top x trigger value for every process, queues and counters.
    - Meanwhile, the trigger values for top x bandwidth were also maintained for every process, queues and counters.
    - Running the tool with `threshold_detection_mode` enabled will thus generate the `threshold_dump` file which can then be used when running the tool in `Auto mode`.

 
3. Auto mode:

    - This mode accepts threshold dump file(output of threshold detection mode) for threshold value.
    - This mode will continuously monitor the CPU stats, counters and handoff queue drops over the period of time. 
    - Meanwhile, the parsed value will be checked against the trigger value(from threshold_dump file) for threshold hit.
    - This monitoring will continue until threshold hits and those monitored dump will be zipped every time.
    - The `window_size` property will determine the number of last x valid zip to be maintained. 
    - The moment it hits, the perf profiling will gets initiated.
    - The critical threads have been identified and analysed as mentioned and the potential root cause report will be generated.
    - This entire process will continue until `consecutive_threshold_exceed_limit` expires.


### Output

The final report contains,

1. Summary report:
    1.   CPU utilization


        -----------------------------------
        Thread Name     | cpu            |
        -----------------------------------
        dpdk_master     | 257.782158136  |
        -----------------------------------
        link_encrypt_1  | 36.9925875371  |
        -----------------------------------
        link_encrypt_0  | 35.4641264231  |
        -----------------------------------
        esp_1           | 35.4054537304  |
        -----------------------------------

        
    2. Handoff Queue drops

    Total drops during the test is provided outside the bracket. And the average drop throughout the test is provided within the braces.



        ---------------------------------------
        Handoff Queue Name  | drops          |
        ---------------------------------------
        vc_queue_ipv4_bh_0  | 332869(83217)  |
        ---------------------------------------
        vc_queue_ipv4_bh_1  | 260869(65217)  |
        ---------------------------------------
        vc_queue_net_sch    | 0(0)           |
        ---------------------------------------
        vc_queue_async_fc   | 0(0)           |
        ---------------------------------------


    3. Counters

    Total drops during the test is provided outside the bracket. And the average drop throughout the test is provided within the braces.


        -------------------------------------------------
        Counter Name              | drops              |
        -------------------------------------------------
        dpdk_sfp2_pstat_ipackets  | 15488408(3872102)  |
        -------------------------------------------------
        dpdk_sfp1_pstat_opackets  | 9991577(2497894)   |
        -------------------------------------------------
        dpdk_sfp1_pstat_ipackets  | 9964850(2491212)   |
        -------------------------------------------------
        dpdk_sfp2_pstat_opackets  | 8734277(2183569)   |
        -------------------------------------------------

2. Detailed report:

    1.   CPU utilization

    This detailed report for CPU utilization will list the top 10 processor consuming threads and their corresponding cpu percent and their perf analysis information. Field `cpu` represents an average system-wide CPU utilization as a percentage. Field 3 represents the perf profile report and field 4 represents the performance counter stat for the corresponding thread. 


        ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        Thread Name     | cpu            | perf Report                                                                                                  | Perf Stat                                                                                       |
        ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        dpdk_master     | 257.782158136  | Report_1                                                                                                     |                                                                                                 |
                        |                | # Children      Self  Command      Shared Object                 Symbol                                      |             291740      dTLB-loads                                                              |
                        |                | # ........  ........  ...........  ............................  ....................................        |                221      dTLB-load-misses          #    0.08% of all dTLB cache hits             |
                        |                | #                                                                                                            |             128088      dTLB-stores                                                             |
                        |                |      1.39%     0.01%  dpdk_master  [kernel.kallsyms]             [k] entry_SYSCALL_64_after_hwframe          |      <not counted>      dTLB-store-misses                                             (0.00%)   |
                        |                |      1.38%     0.04%  dpdk_master  [kernel.kallsyms]             [k] do_syscall_64                           |      <not counted>      iTLB-loads                                                              |
                        |                |      1.34%     0.01%  dpdk_master  [kernel.kallsyms]             [k] sys_nanosleep                           |      <not counted>      iTLB-load-misses                                                        |
                        |                |      1.25%     0.01%  dpdk_master  [kernel.kallsyms]             [k] hrtimer_nanosleep                       |      <not counted>      LLC-loads                                                               |
                        |                |      1.23%     0.02%  dpdk_master  [kernel.kallsyms]             [k] do_nanosleep                            |      <not counted>      LLC-load-misses                                                         |
                        |                |      0.97%     0.02%  dpdk_master  [kernel.kallsyms]             [k] schedule                                |      <not counted>      LLC-stores                                                              |
                        |                |      0.93%     0.12%  dpdk_master  [kernel.kallsyms]             [k] __schedule                              |      <not counted>      LLC-store-misses                                                        |
                        |                |      0.66%     0.66%  dpdk_master  edged                         [.] edged_vnf_ether_input                   |    <not supported>      LLC-prefetch-misses                                                     |
                        |                |      0.65%     0.65%  dpdk_master  edged                         [.] dpdk_do_rx_once                         |                 72      page-faults                                                             |
                        |                |      0.51%     0.51%  dpdk_master  edged                         [.] dpdk_do_tx_once                         |      <not counted>      cache-misses                                                            |
                        |                |      0.33%     0.33%  dpdk_master  libc-2.19.so                  [.] __memcmp_sse4_1                         |      <not counted>      cache-references                                                        |
                        |                |      0.32%     0.32%  dpdk_master  librte_pmd_i40e.so.20.0       [.] i40e_recv_pkts_vec_avx2                 |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
                        |                | Report_2                                                                                                     |                                                                                                 |
                        |                | # Children      Self  Command      Shared Object                 Symbol                                      |                                                                                                 |
                        |                | # ........  ........  ...........  ............................  ....................................        |                                                                                                 |
                        |                | #                                                                                                            |                                                                                                 |
                        |                |      1.64%     0.00%  dpdk_master  [kernel.kallsyms]             [k] entry_SYSCALL_64_after_hwframe          |                                                                                                 |
                        |                |      1.54%     0.02%  dpdk_master  [kernel.kallsyms]             [k] do_nanosleep                            |                                                                                                 |
                        |                |      1.30%     0.01%  dpdk_master  [kernel.kallsyms]             [k] schedule                                |                                                                                                 |
                        |                |      1.28%     0.14%  dpdk_master  [kernel.kallsyms]             [k] __schedule                              |                                                                                                 |
                        |                |      0.64%     0.64%  dpdk_master  edged                         [.] dpdk_do_rx_once                         |                                                                                                 |
                        |                |      0.64%     0.64%  dpdk_master  edged                         [.] edged_vnf_ether_input                   |                                                                                                 |
                        |                |      0.37%     0.00%  dpdk_master  [kernel.kallsyms]             [k] perf_swevent_event                      |                                                                                                 |
                        |                |      0.36%     0.00%  dpdk_master  [kernel.kallsyms]             [k] __perf_event_overflow                   |                                                                                                 |
                        |                |      0.35%     0.00%  dpdk_master  [kernel.kallsyms]             [k] perf_event_output_forward               |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
                        |                |        .         .                     .                          .                                          |                                                                                                 |
        ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        link_encrypt_1  |  36.9925875371 |        .         .                     .                          .                                          |                                     .                                                           |
                        |                |        .         .                     .                          .                                          |                                     .                                                           |
                        |                |                                                                                                              |                                                                                                 |
        ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
            .           |       .        |        .         .                     .                          .                                          |                                     .                                                           |
            .           |       .        |        .         .                     .                          .                                          |                                     .                                                           |
                        |                |                                                                                                              |                                                                                                 |
        ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


    2.   Handoff Queue drops

    This detailed report for Handoff Queue drops will list the top 10 queues with their drops and their perf analysis information. Field 2 represents total drops(outside the bracket) and average drop(within the braces). Field 3 represents the perf profile report and field 4 represents the performance counter stat for the corresponding thread. 
    
        --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        Handoff Queue Name  | drops          | perf Report                                                                                                  | Perf Stat                                                                                       |
        --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        vc_queue_ipv4_bh_0  | 332869(83217)  | Report_1                                                                                                     |                                                                                                 |
                            |                | # Children      Self  Command    Shared Object            Symbol                                             |              18831      dTLB-loads                                                              |
                            |                | # ........  ........  .........  .......................  .................................................  |                214      dTLB-load-misses          #    1.14% of all dTLB cache hits             |
                            |                | #                                                                                                            |             136060      dTLB-stores                                                             |
                            |                |      3.03%     3.03%  ipv4_bh_0  libpthread-2.19.so       [.] pthread_rwlock_rdlock                          |                145      dTLB-store-misses                                             (89.52%)  |
                            |                |      1.97%     0.07%  ipv4_bh_0  [kernel.kallsyms]        [k] entry_SYSCALL_64_after_hwframe                 |                284      iTLB-loads                                                    (26.77%)  |
                            |                |      1.89%     0.06%  ipv4_bh_0  [kernel.kallsyms]        [k] do_syscall_64                                  |      <not counted>      iTLB-load-misses                                              (0.00%)   |
                            |                |      1.73%     0.01%  ipv4_bh_0  [kernel.kallsyms]        [k] sys_futex                                      |      <not counted>      LLC-loads                                                               |
                            |                |      1.72%     0.00%  ipv4_bh_0  [kernel.kallsyms]        [k] do_futex                                       |      <not counted>      LLC-load-misses                                                         |
                            |                |      1.27%     1.27%  ipv4_bh_0  libpthread-2.19.so       [.] pthread_rwlock_unlock                          |      <not counted>      LLC-stores                                                              |
                            |                |      0.89%     0.24%  ipv4_bh_0  [kernel.kallsyms]        [k] futex_wake                                     |      <not counted>      LLC-store-misses                                                        |
                            |                |      0.70%     0.70%  ipv4_bh_0  libpthread-2.19.so       [.] pthread_mutex_lock                             |    <not supported>      LLC-prefetch-misses                                                     |
                            |                |      0.69%     0.04%  ipv4_bh_0  [kernel.kallsyms]        [k] futex_wait                                     |                 75      page-faults                                                             |
                            |                |      0.53%     0.53%  ipv4_bh_0  edged                    [.] vcmp_fc_hold                                   |      <not counted>      cache-misses                                                            |
                            |                |      0.50%     0.50%  ipv4_bh_0  edged                    [.] vc_edge_route_hold                             |      <not counted>      cache-references                                                        |
                            |                |      0.44%     0.44%  ipv4_bh_0  edged                    [.] vc_ht_lookup                                   |                                                                                                 |
                            |                |        .         .                 .                             .                                           |                                                                                                 |
                            |                |        .         .                 .                             .                                           |                                                                                                 |  
                            |                |                                                                                                              |                                                                                                 |
                            |                | Report_2                                                                                                     |                                                                                                 |
                            |                | # Children      Self  Command    Shared Object            Symbol                                             |                                                                                                 |
                            |                | # ........  ........  .........  .......................  .................................................  |                                                                                                 |
                            |                | #                                                                                                            |                                                                                                 |
                            |                |      2.25%     2.25%  ipv4_bh_0  libpthread-2.19.so       [.] pthread_rwlock_rdlock                          |                                                                                                 |
                            |                |      1.97%     0.04%  ipv4_bh_0  [kernel.kallsyms]        [k] entry_SYSCALL_64_after_hwframe                 |                                                                                                 |
                            |                |      1.95%     0.12%  ipv4_bh_0  [kernel.kallsyms]        [k] do_syscall_64                                  |                                                                                                 |
                            |                |      1.71%     0.03%  ipv4_bh_0  [kernel.kallsyms]        [k] sys_futex                                      |                                                                                                 |
                            |                |      1.68%     0.02%  ipv4_bh_0  [kernel.kallsyms]        [k] do_futex                                       |                                                                                                 |
                            |                |      1.19%     1.19%  ipv4_bh_0  libpthread-2.19.so       [.] pthread_rwlock_unlock                          |                                                                                                 |
                            |                |      0.80%     0.03%  ipv4_bh_0  [kernel.kallsyms]        [k] futex_wait                                     |                                                                                                 |
                            |                |        .         .                     .                          .                                          |                                                                                                 |
                            |                |        .         .                     .                          .                                          |                                                                                                 |
                            |                |        .         .                     .                          .                                          |                                                                                                 |
        --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        vc_queue_ipv4_bh_1  | 260869(65217)  |        .         .                     .                          .                                          |                                                                                                 |
                            |                |        .         .                     .                          .                                          |                                                                                                 |
                            |                |                                                                                                              |                                                                                                 |
        --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
                .           |       .        |        .         .                     .                          .                                          |                                                                                                 |
                .           |       .        |        .         .                     .                          .                                          |                                                                                                 |
                            |                |                                                                                                              |                                                                                                 |
        --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    3. Counters

    Total drops during the test is provided outside the bracket. And the average drop throughout the test is provided within the braces.

        --------------------------------------------------------
        Counter Name                     | drops              |
        --------------------------------------------------------
        dpdk_sfp2_pstat_ipackets         | 15488408(3872102)  |
        --------------------------------------------------------
        dpdk_sfp1_pstat_opackets         | 9991577(2497894)   |
        --------------------------------------------------------
        dpdk_sfp1_pstat_ipackets         | 9964850(2491212)   |
        --------------------------------------------------------
        dpdk_sfp2_pstat_opackets         | 8734277(2183569)   |
        --------------------------------------------------------
        dpdk_ge3_pstat_ipackets          | 200(50)            |
        --------------------------------------------------------
        dpdk_br-network1_pstat_ipackets  | 100(25)            |
        --------------------------------------------------------
        dpdk_ge3_kni_rx_copy_packets     | 74(18)             |
        --------------------------------------------------------
        dpdk_ge3_kni_rx_packets          | 72(18)             |
        --------------------------------------------------------
        dpdk_ge5_pstat_ipackets          | 53(13)             |
        --------------------------------------------------------
        dpdk_ge6_pstat_ipackets          | 53(13)             |
        --------------------------------------------------------

3. Perf sched latency report:
    1.   Based on the key `switch`:

    The entire latency report is sorted based on the field `Switches` and displayed.

        -------------------------------------------------------------------------------------------------------------------------------------------------------------
        Task           | dpdk_master  | ipv4_bh_bottom  | perf      | vcmp_tx  | link_select  | net_sch  | link_encrypt_1  | esp_0    | esp_1    | link_encrypt_0  |
        -------------------------------------------------------------------------------------------------------------------------------------------------------------
        Switches       | 130259.5     | 67614.5         | 58155.0   | 53426.0  | 36577.0      | 29866.5  | 28684.5         | 28674.5  | 28668.5  | 28657.5         |
        -------------------------------------------------------------------------------------------------------------------------------------------------------------
        Maximum delay  | 0.1105       | 0.4405          | 34.838    | 0.4865   | 0.2155       | 0.1795   | 0.22            | 0.222    | 0.297    | 0.4025          |
        -------------------------------------------------------------------------------------------------------------------------------------------------------------
        Runtime        | 0.0          | 0.0             | 1747.045  | 0.0      | 0.0          | 0.0      | 0.0             | 0.0      | 0.0      | 0.0             |
        -------------------------------------------------------------------------------------------------------------------------------------------------------------
        Average delay  | 0.002        | 0.001           | 0.027     | 0.001    | 0.001        | 0.0015   | 0.0015          | 0.0015   | 0.0015   | 0.0015          |
        -------------------------------------------------------------------------------------------------------------------------------------------------------------

    2.   Based on the key `Average delay`:

    Here the latency report is sorted based on the field `Average delay`.

        ------------------------------------------------------------------------------------------------------------------------------------
        Task           | kworker/7  | vnfd.py  | bfdd   | pimd   | grep    | log_archive  | awk    | watchfrr  | conntrackd  | kworker/6  |
        ------------------------------------------------------------------------------------------------------------------------------------
        Switches       | 15.0       | 83.0     | 54.0   | 189.0  | 481.0   | 2329.0       | 367.0  | 102.5     | 219.5       | 27.0       |
        ------------------------------------------------------------------------------------------------------------------------------------
        Maximum delay  | 77.3505    | 8.5625   | 1.101  | 6.128  | 17.502  | 15.6395      | 8.833  | 3.4395    | 0.7265      | 1.429      |
        ------------------------------------------------------------------------------------------------------------------------------------
        Runtime        | 0.2385     | 0.5565   | 0.216  | 0.576  | 5.799   | 12.4075      | 2.972  | 0.441     | 1.002       | 0.736      |
        ------------------------------------------------------------------------------------------------------------------------------------
        Average delay  | 5.23       | 0.269    | 0.183  | 0.182  | 0.134   | 0.1325       | 0.126  | 0.123     | 0.1165      | 0.1085     |
        ------------------------------------------------------------------------------------------------------------------------------------

### Getting started
1. Install the tool in the edge device under test
2. How to run 
  
   - Normal Mode:
   
          python index.py [-s]
          
   - Threshold detection mode
   
          python index.py -T [-o <output file>], [-d n]

   - Auto mode:
   
          python index.py -A -i <threshold_dump> [-m value | bandwidth], [-w n], [-l n], [-s]
              
   - Offline mode:
   
          python index.py -F <zip file>
              
