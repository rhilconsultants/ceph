# MGR Modules 

This exercise will walk you through the different mgr modules. You could see the Dashboard (that has turned into a RW and can be used as operational GUI), the Prometheus module the integrated with `cephmetrics` (now part of the deployment out of the box) and the balancer module. 

**Note: ** ‘cephmetrics’ and Dashboard will be presented  by the instructor as they are more a GUI exercises 

## Balancer 

Login to the mon server, use `ceph osd df` command to verify pg distribution: 

```bash
$ ceph osd df 
ID CLASS WEIGHT  REWEIGHT SIZE    RAW USE DATA    OMAP META  AVAIL   %USE VAR  PGS STATUS 
 0   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  36     up 
 3   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  44     up 
 2   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  39     up 
 5   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  41     up 
 1   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  43     up 
 4   hdd 0.04790  1.00000  49 GiB 1.0 GiB  28 MiB  0 B 1 GiB  48 GiB 2.10 1.00  37     up 
                    TOTAL 294 GiB 6.2 GiB 169 MiB  0 B 6 GiB 288 GiB 2.10                 
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

We see the the CRUSH map divided the pgs across OSDs unevenly, which means that some of the OSDs will have more space consumed and more IO as the pg number affect this metrics directly. This can often create performance problems as there is one OSD that can affect akk the others (Ceph is a distributed system). 

As you see, in the `PGS` tab we can see pg num for each OSD have close values, but it’s not a perfect distribution, enable balancer module: 

```bash
$ ceph mgr module enable balancer 
```

Start balancer module: 

```bash
$ ceph balancer on 
```

Set minimum client luminous field to true, then enable upmap mode: 

```bash
$ ceph osd set-require-min-compat-client luminous
$ ceph balancer mode upmap
```

The upmap mode will switch between pgs in a given OSD to create a better distribution of pgs across OSDs. 

Set balancer module off and on:

```bash
$ Ceph balancer off && ceph balancer on 
```

Use `ceph osd df` command to verify pgs are evenly distributed: 

```bash
$ceph osd df 

ID CLASS WEIGHT  REWEIGHT SIZE    RAW USE DATA    OMAP META  AVAIL   %USE VAR  PGS STATUS 
 0   hdd 0.04790  1.00000  49 GiB 1.0 GiB  29 MiB  0 B 1 GiB  48 GiB 2.10 1.00  40     up 
 3   hdd 0.04790  1.00000  49 GiB 1.0 GiB  29 MiB  0 B 1 GiB  48 GiB 2.10 1.00  41     up 
 2   hdd 0.04790  1.00000  49 GiB 1.0 GiB  30 MiB  0 B 1 GiB  48 GiB 2.10 1.00  39     up 
 5   hdd 0.04790  1.00000  49 GiB 1.0 GiB  29 MiB  0 B 1 GiB  48 GiB 2.10 1.00  41     up 
 1   hdd 0.04790  1.00000  49 GiB 1.0 GiB  29 MiB  0 B 1 GiB  48 GiB 2.10 1.00  40     up 
 4   hdd 0.04790  1.00000  49 GiB 1.0 GiB  29 MiB  0 B 1 GiB  48 GiB 2.10 1.00  40     up 
                    TOTAL 294 GiB 6.2 GiB 176 MiB  0 B 6 GiB 288 GiB 2.10                 
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

Now we see that each OSD has the same amount of pgs. 


