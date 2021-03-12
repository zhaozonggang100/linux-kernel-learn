```c
file：arch/arm64/boot/dts/qcom/sm8150.dtsi

2319     ufshc_mem: ufshc@1d84000 {
2320         compatible = "qcom,ufshc";
2321         reg = <0x1d84000 0x2500>;
2322         interrupts = <0 265 0>;
2323         phys = <&ufsphy_mem>;
2324         phy-names = "ufsphy";
2325         ufs-qcom-crypto = <&ufs_ice>;
2326
2327         lanes-per-direction = <2>;
2328         dev-ref-clk-freq = <0>; /* 19.2 MHz */
2329
2330         clock-names =
2331             "core_clk",
2332             "bus_aggr_clk",
2333             "iface_clk",
2334             "core_clk_unipro",
2335             "core_clk_ice",
2336             "ref_clk",
2337             "tx_lane0_sync_clk",
2338             "rx_lane0_sync_clk",
2339             "rx_lane1_sync_clk";
2340         clocks =
2341             <&clock_gcc GCC_UFS_PHY_AXI_CLK>,
2342             <&clock_gcc GCC_AGGRE_UFS_PHY_AXI_CLK>,
2343             <&clock_gcc GCC_UFS_PHY_AHB_CLK>,
2344             <&clock_gcc GCC_UFS_PHY_UNIPRO_CORE_CLK>,
2345             <&clock_gcc GCC_UFS_PHY_ICE_CORE_CLK>,
2346             <&clock_rpmh RPMH_CXO_CLK>,
2347             <&clock_gcc GCC_UFS_PHY_TX_SYMBOL_0_CLK>,
2348             <&clock_gcc GCC_UFS_PHY_RX_SYMBOL_0_CLK>,
2349             <&clock_gcc GCC_UFS_PHY_RX_SYMBOL_1_CLK>;
2350         freq-table-hz =
2351             <37500000 300000000>,
2352             <0 0>,
2353             <0 0>,
2354             <37500000 300000000>,
2355             <37500000 300000000>,
2356             <0 0>,
2357             <0 0>,
2358             <0 0>,
2359             <0 0>;
2360
2361         qcom,msm-bus,name = "ufshc_mem";
2362         qcom,msm-bus,num-cases = <26>;
2363         qcom,msm-bus,num-paths = <2>;
2364         qcom,msm-bus,vectors-KBps =
2365         /*
2366          * During HS G3 UFS runs at nominal voltage corner, vote
2367          * higher bandwidth to push other buses in the data path
2368          * to run at nominal to achieve max throughput.
2369          * 4GBps pushes BIMC to run at nominal.
2370          * 200MBps pushes CNOC to run at nominal.
2371          * Vote for half of this bandwidth for HS G3 1-lane.
2372          * For max bandwidth, vote high enough to push the buses
2373          * to run in turbo voltage corner.
2374          */
2375         <123 512 0 0>, <1 757 0 0>,          /* No vote */
2376         <123 512 922 0>, <1 757 1000 0>,     /* PWM G1 */
2377         <123 512 1844 0>, <1 757 1000 0>,    /* PWM G2 */
2378         <123 512 3688 0>, <1 757 1000 0>,    /* PWM G3 */
2379         <123 512 7376 0>, <1 757 1000 0>,    /* PWM G4 */
2380         <123 512 1844 0>, <1 757 1000 0>,    /* PWM G1 L2 */
2381         <123 512 3688 0>, <1 757 1000 0>,    /* PWM G2 L2 */
2382         <123 512 7376 0>, <1 757 1000 0>,    /* PWM G3 L2 */
2383         <123 512 14752 0>, <1 757 1000 0>,   /* PWM G4 L2 */
2384         <123 512 127796 0>, <1 757 1000 0>,  /* HS G1 RA */
2385         <123 512 255591 0>, <1 757 1000 0>,  /* HS G2 RA */
2386         <123 512 2097152 0>, <1 757 102400 0>,  /* HS G3 RA */
2387         <123 512 4194304 0>, <1 757 204800 0>,  /* HS G4 RA */
2388         <123 512 255591 0>, <1 757 1000 0>,  /* HS G1 RA L2 */
2389         <123 512 511181 0>, <1 757 1000 0>,  /* HS G2 RA L2 */
2390         <123 512 4194304 0>, <1 757 204800 0>, /* HS G3 RA L2 */
2391         <123 512 8388608 0>, <1 757 409600 0>, /* HS G4 RA L2 */
2392         <123 512 149422 0>, <1 757 1000 0>,  /* HS G1 RB */
2393         <123 512 298189 0>, <1 757 1000 0>,  /* HS G2 RB */
2394         <123 512 2097152 0>, <1 757 102400 0>,  /* HS G3 RB */
2395         <123 512 4194304 0>, <1 757 204800 0>,  /* HS G4 RB */
2396         <123 512 298189 0>, <1 757 1000 0>,  /* HS G1 RB L2 */
2397         <123 512 596378 0>, <1 757 1000 0>,  /* HS G2 RB L2 */
2398         /* As UFS working in HS G3 RB L2 mode, aggregated
2399          * bandwidth (AB) should take care of providing
2400          * optimum throughput requested. However, as tested,
2401          * in order to scale up CNOC clock, instantaneous
2402          * bindwidth (IB) needs to be given a proper value too.
2403          */
2404         <123 512 4194304 0>, <1 757 204800 409600>, /* HS G3 RB L2 */
2405         <123 512 8388608 0>, <1 757 409600 409600>, /* HS G4 RB L2 */
2406         <123 512 7643136 0>, <1 757 307200 0>; /* Max. bandwidth */
2407
2408         qcom,bus-vector-names = "MIN",
2409         "PWM_G1_L1", "PWM_G2_L1", "PWM_G3_L1", "PWM_G4_L1",
2410         "PWM_G1_L2", "PWM_G2_L2", "PWM_G3_L2", "PWM_G4_L2",
2411         "HS_RA_G1_L1", "HS_RA_G2_L1", "HS_RA_G3_L1", "HS_RA_G4_L1",
2412         "HS_RA_G1_L2", "HS_RA_G2_L2", "HS_RA_G3_L2", "HS_RA_G4_L2",
2413         "HS_RB_G1_L1", "HS_RB_G2_L1", "HS_RB_G3_L1", "HS_RB_G4_L1",
2414         "HS_RB_G1_L2", "HS_RB_G2_L2", "HS_RB_G3_L2", "HS_RB_G4_L2",
2415         "MAX";
2416
2417         /* PM QoS */
2418         qcom,pm-qos-cpu-groups = <0x0f 0xf0>;
2419         qcom,pm-qos-cpu-group-latency-us = <44 44>;
2420         qcom,pm-qos-default-cpu = <0>;
2421
2422         pinctrl-names = "dev-reset-assert", "dev-reset-deassert";
2423         pinctrl-0 = <&ufs_dev_reset_assert>;
2424         pinctrl-1 = <&ufs_dev_reset_deassert>;
2425
2426         resets = <&clock_gcc GCC_UFS_PHY_BCR>;
2427         reset-names = "core_reset";
2428
2429         status = "disabled";
2430     };
```

