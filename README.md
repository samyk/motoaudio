# motoaudio
Inspecting the Moto Audio application running on Motorola Android devices

## OS

Found the (purported) OS [here](https://mirrors.lolinet.com/firmware/moto/potter/official/RETUS/), currently using [XT1687_POTTER_RETUS_8.1.0_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.xml.zip](https://mirrors.lolinet.com/firmware/moto/potter/official/RETUS/XT1687_POTTER_RETUS_8.1.0_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.xml.zip) (1.4GB).

## OS Extraction

Unpack the zip file
`unzip XT1687_POTTER_RETUS_8.1.0_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.xml.zip`
```
Archive:  XT1687_POTTER_RETUS_8.1.0_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.xml.zip
  inflating: NON-HLOS.bin
  inflating: recovery.img
  inflating: fsg.mbn
  inflating: gpt.bin
  inflating: boot.img
  inflating: system.img_sparsechunk.0
  inflating: system.img_sparsechunk.1
  inflating: system.img_sparsechunk.2
  inflating: system.img_sparsechunk.3
  inflating: system.img_sparsechunk.4
  inflating: system.img_sparsechunk.5
  inflating: system.img_sparsechunk.6
  inflating: system.img_sparsechunk.7
  inflating: system.img_sparsechunk.8
  inflating: adspso.bin
  inflating: oem.img
  inflating: logo.bin
  inflating: bootloader.img
  inflating: flashfile.xml
  inflating: servicefile.xml
 extracting: slcf_rev_d_default_v1.0.nvm
 extracting: regulatory_info_default.png
  inflating: POTTER_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.info.txt
```

Now we have some Android sparse images and boot images for the file system.
```sh
file NON-HLOS.bin recovery.img fsg.mbn gpt.bin boot.img system.img_sparsechunk.0 system.img_sparsechunk.1 system.img_sparsechunk.2 system.img_sparsechunk.3 system.img_sparsechunk.4 system.img_sparsechunk.5 system.img_sparsechunk.6 system.img_sparsechunk.7 system.img_sparsechunk.8 adspso.bin oem.img logo.bin bootloader.img flashfile.xml servicefile.xml slcf_rev_d_default_v1.0.nvm regulatory_info_default.png POTTER_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.info.txt
NON-HLOS.bin:                                                                 Android sparse image, version: 1.0, Total of 25600 4096-byte output blocks in 128 input chunks.
recovery.img:                                                                 Android bootimg, kernel, ramdisk, page size: 2048, cmdline (console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom user_debug=30 m)
fsg.mbn:                                                                      Linux rev 1.0 ext2 filesystem data, UUID=e4a4f807-109f-5459-8138-e744bc88c397, volume name "FSG" (extents) (large files)
gpt.bin:                                                                      data
boot.img:                                                                     Android bootimg, kernel, ramdisk, page size: 2048, cmdline (console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom user_debug=30 m)
system.img_sparsechunk.0:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 49 input chunks.
system.img_sparsechunk.1:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 156 input chunks.
system.img_sparsechunk.2:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 1368 input chunks.
system.img_sparsechunk.3:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 83 input chunks.
system.img_sparsechunk.4:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 141 input chunks.
system.img_sparsechunk.5:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 87 input chunks.
system.img_sparsechunk.6:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 1144 input chunks.
system.img_sparsechunk.7:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 1446 input chunks.
system.img_sparsechunk.8:                                                     Android sparse image, version: 1.0, Total of 884769 4096-byte output blocks in 164 input chunks.
adspso.bin:                                                                   Linux rev 1.0 ext4 filesystem data, UUID=af32c008-2a39-7e5b-a5dc-201456d93103, volume name "dsp" (extents) (large files)
oem.img:                                                                      Android sparse image, version: 1.0, Total of 167969 4096-byte output blocks in 67 input chunks.
logo.bin:                                                                     data
bootloader.img:                                                               data
flashfile.xml:                                                                XML 1.0 document text, ASCII text
servicefile.xml:                                                              XML 1.0 document text, ASCII text
slcf_rev_d_default_v1.0.nvm:                                                  empty
regulatory_info_default.png:                                                  empty
POTTER_OPS28.85-17-6-2_cid50_subsidy-DEFAULT_regulatory-DEFAULT_CFC.info.txt: ASCII text
```

### Finding Moto Audio application

There's a message `Processing analytics data...` that we're looking for so let's see if it's in pain sight first, using my [g](https://github.com/samyk/samytools/blob/master/g) tool for fast binary grepping.

```sh
g -aU 'Processing analytics data'
Binary file system.img_sparsechunk.4 matches
```

Nice, so it's in that chunk. We can reconstruct all the sparse images into a standard raw image via [android-simg2img](https://github.com/anestisb/android-simg2img) but since these are large files and likely multiple layers of archival, it may just be easier to take a look inside the file.

Let's find the index in the file where it exists.
```sh
perl -0777 -ne 'print index($_,"Processing analytics data")' system.img_sparsechunk.4
262616254
```

I suspect there are ARM [ELF binaries](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) nearby, let's grab the string and move forward in the file from the start until we find the last ELF header before the string, then extract to the string plus five megabytes.

```sh
perl -0777 -ne '$data = $_; $strind = index($data, "Processing analytics data"); $lastind = $ind while ($ind = index($_, "\x7FELF", $lastind+1)) < $strind; print substr($data, $lastind, $strind-$lastind+1024*1024*5);' system.img_sparsechunk.4 > bin1
```

Now make sure it looks like a binary.
```sh
file bin1
bin1: ELF 32-bit LSB shared object ARM, EABI5 version 1 (GNU/Linux), dynamically linked, BuildID[sha1]=b665e3331110f742fa922a482b82b8d96f6c97d2, stripped
```

Is there other data in here? Let's extract random binaries via binwalk.
```sh
mkdir bin2 && cd bin2 && binwalk -e ../bin1

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 32-bit LSB shared object, ARM, version 1 (GNU/Linux)
4351          0x10FF          Unix path: /target/product/potter/dex_bootjars/system/framework/boot.art --dex-file=out/target/product/potter/obj/APPS/AttPhoneExt_intermed
48244         0xBC74          Unix path: /android/internal/telephony/CommandException$Error;
63561         0xF849          Unix path: /android/internal/telephony/PhoneInternalInterface;
73728         0x12000         Zip archive data, at least v1.0 to extract, compressed size: 287, uncompressed size: 287, name: res/drawable-hdpi-v4/abc_ab_share_pack_mtrl_alpha.9.png
74111         0x1217F         Zip archive data, at least v1.0 to extract, compressed size: 227, uncompressed size: 227, name: res/drawable-hdpi-v4/abc_btn_check_to_on_mtrl_000.png
74427         0x122BB         Zip archive data, at least v1.0 to extract, compressed size: 404, uncompressed size: 404, name: res/drawable-hdpi-v4/abc_btn_check_to_on_mtrl_015.png
74920         0x124A8         Zip archive data, at least v1.0 to extract, compressed size: 464, uncompressed size: 464, name: res/drawable-hdpi-v4/abc_btn_radio_to_on_mtrl_000.png
75476         0x126D4         Zip archive data, at least v1.0 to extract, compressed size: 563, uncompressed size: 563, name: res/drawable-hdpi-v4/abc_btn_radio_to_on_mtrl_015.png
76131         0x12963         Zip archive data, at least v1.0 to extract, compressed size: 1548, uncompressed size: 1548, name: res/drawable-hdpi-v4/abc_btn_switch_to_on_mtrl_00001.9.png
77776         0x12FD0         Zip archive data, at least v1.0 to extract, compressed size: 1748, uncompressed size: 1748, name: res/drawable-hdpi-v4/abc_btn_switch_to_on_mtrl_00012.9.png
79620         0x13704         Zip archive data, at least v1.0 to extract, compressed size: 229, uncompressed size: 229, name: res/drawable-hdpi-v4/abc_cab_background_top_mtrl_alpha.9.png
79945         0x13849         Zip archive data, at least v1.0 to extract, compressed size: 171, uncompressed size: 171, name: res/drawable-hdpi-v4/abc_ic_commit_search_api_mtrl_alpha.png
80215         0x13957         Zip archive data, at least v1.0 to extract, compressed size: 202, uncompressed size: 202, name: res/drawable-hdpi-v4/abc_ic_menu_copy_mtrl_am_alpha.png
80510         0x13A7E         Zip archive data, at least v1.0 to extract, compressed size: 404, uncompressed size: 404, name: res/drawable-hdpi-v4/abc_ic_menu_cut_mtrl_alpha.png
81004         0x13C6C         Zip archive data, at least v1.0 to extract, compressed size: 226, uncompressed size: 226, name: res/drawable-hdpi-v4/abc_ic_menu_paste_mtrl_am_alpha.png
81322         0x13DAA         Zip archive data, at least v1.0 to extract, compressed size: 215, uncompressed size: 215, name: res/drawable-hdpi-v4/abc_ic_menu_selectall_mtrl_alpha.png
81631         0x13EDF         Zip archive data, at least v1.0 to extract, compressed size: 389, uncompressed size: 389, name: res/drawable-hdpi-v4/abc_ic_menu_share_mtrl_alpha.png
82109         0x140BD         Zip archive data, at least v1.0 to extract, compressed size: 263, uncompressed size: 263, name: res/drawable-hdpi-v4/abc_ic_star_black_16dp.png
82455         0x14217         Zip archive data, at least v1.0 to extract, compressed size: 522, uncompressed size: 522, name: res/drawable-hdpi-v4/abc_ic_star_black_36dp.png
83062         0x14476         Zip archive data, at least v1.0 to extract, compressed size: 668, uncompressed size: 668, name: res/drawable-hdpi-v4/abc_ic_star_black_48dp.png
83816         0x14768         Zip archive data, at least v1.0 to extract, compressed size: 197, uncompressed size: 197, name: res/drawable-hdpi-v4/abc_ic_star_half_black_16dp.png
84101         0x14885         Zip archive data, at least v1.0 to extract, compressed size: 328, uncompressed size: 328, name: res/drawable-hdpi-v4/abc_ic_star_half_black_36dp.png
84520         0x14A28         Zip archive data, at least v1.0 to extract, compressed size: 431, uncompressed size: 431, name: res/drawable-hdpi-v4/abc_ic_star_half_black_48dp.png
85039         0x14C2F         Zip archive data, at least v1.0 to extract, compressed size: 167, uncompressed size: 167, name: res/drawable-hdpi-v4/abc_list_divider_mtrl_alpha.9.png
85299         0x14D33         Zip archive data, at least v1.0 to extract, compressed size: 244, uncompressed size: 244, name: res/drawable-hdpi-v4/abc_list_focused_holo.9.png
85628         0x14E7C         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-hdpi-v4/abc_list_longpressed_holo.9.png
85928         0x14FA8         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-hdpi-v4/abc_list_pressed_holo_dark.9.png
86232         0x150D8         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-hdpi-v4/abc_list_pressed_holo_light.9.png
86536         0x15208         Zip archive data, at least v1.0 to extract, compressed size: 239, uncompressed size: 239, name: res/drawable-hdpi-v4/abc_list_selector_disabled_holo_dark.9.png
86875         0x1535B         Zip archive data, at least v1.0 to extract, compressed size: 240, uncompressed size: 240, name: res/drawable-hdpi-v4/abc_list_selector_disabled_holo_light.9.png
87216         0x154B0         Zip archive data, at least v1.0 to extract, compressed size: 817, uncompressed size: 817, name: res/drawable-hdpi-v4/abc_menu_hardkey_panel_mtrl_mult.9.png
88129         0x15841         Zip archive data, at least v1.0 to extract, compressed size: 1256, uncompressed size: 1256, name: res/drawable-hdpi-v4/abc_popup_background_mtrl_mult.9.png
89480         0x15D88         Zip archive data, at least v1.0 to extract, compressed size: 201, uncompressed size: 201, name: res/drawable-hdpi-v4/abc_scrubber_control_off_mtrl_alpha.png
89777         0x15EB1         Zip archive data, at least v1.0 to extract, compressed size: 196, uncompressed size: 196, name: res/drawable-hdpi-v4/abc_scrubber_control_to_pressed_mtrl_000.png
90076         0x15FDC         Zip archive data, at least v1.0 to extract, compressed size: 272, uncompressed size: 272, name: res/drawable-hdpi-v4/abc_scrubber_control_to_pressed_mtrl_005.png
90452         0x16154         Zip archive data, at least v1.0 to extract, compressed size: 214, uncompressed size: 214, name: res/drawable-hdpi-v4/abc_scrubber_primary_mtrl_alpha.9.png
90762         0x1628A         Zip archive data, at least v1.0 to extract, compressed size: 201, uncompressed size: 201, name: res/drawable-hdpi-v4/abc_scrubber_track_mtrl_alpha.9.png
91057         0x163B1         Zip archive data, at least v1.0 to extract, compressed size: 368, uncompressed size: 368, name: res/drawable-hdpi-v4/abc_spinner_mtrl_am_alpha.9.png
91516         0x1657C         Zip archive data, at least v1.0 to extract, compressed size: 538, uncompressed size: 538, name: res/drawable-hdpi-v4/abc_switch_track_mtrl_alpha.9.png
92146         0x167F2         Zip archive data, at least v1.0 to extract, compressed size: 199, uncompressed size: 199, name: res/drawable-hdpi-v4/abc_tab_indicator_mtrl_alpha.9.png
92439         0x16917         Zip archive data, at least v1.0 to extract, compressed size: 277, uncompressed size: 277, name: res/drawable-hdpi-v4/abc_text_select_handle_left_mtrl_dark.png
92817         0x16A91         Zip archive data, at least v1.0 to extract, compressed size: 277, uncompressed size: 277, name: res/drawable-hdpi-v4/abc_text_select_handle_left_mtrl_light.png
93193         0x16C09         Zip archive data, at least v1.0 to extract, compressed size: 398, uncompressed size: 398, name: res/drawable-hdpi-v4/abc_text_select_handle_middle_mtrl_dark.png
93694         0x16DFE         Zip archive data, at least v1.0 to extract, compressed size: 396, uncompressed size: 396, name: res/drawable-hdpi-v4/abc_text_select_handle_middle_mtrl_light.png
94192         0x16FF0         Zip archive data, at least v1.0 to extract, compressed size: 263, uncompressed size: 263, name: res/drawable-hdpi-v4/abc_text_select_handle_right_mtrl_dark.png
94555         0x1715B         Zip archive data, at least v1.0 to extract, compressed size: 262, uncompressed size: 262, name: res/drawable-hdpi-v4/abc_text_select_handle_right_mtrl_light.png
94918         0x172C6         Zip archive data, at least v1.0 to extract, compressed size: 192, uncompressed size: 192, name: res/drawable-hdpi-v4/abc_textfield_activated_mtrl_alpha.9.png
95208         0x173E8         Zip archive data, at least v1.0 to extract, compressed size: 198, uncompressed size: 198, name: res/drawable-hdpi-v4/abc_textfield_default_mtrl_alpha.9.png
95502         0x1750E         Zip archive data, at least v1.0 to extract, compressed size: 182, uncompressed size: 182, name: res/drawable-hdpi-v4/abc_textfield_search_activated_mtrl_alpha.9.png
95790         0x1762E         Zip archive data, at least v1.0 to extract, compressed size: 182, uncompressed size: 182, name: res/drawable-hdpi-v4/abc_textfield_search_default_mtrl_alpha.9.png
96074         0x1774A         Zip archive data, at least v1.0 to extract, compressed size: 7455, uncompressed size: 7455, name: res/drawable-hdpi-v4/balanced.png
103599        0x194AF         Zip archive data, at least v1.0 to extract, compressed size: 7179, uncompressed size: 7179, name: res/drawable-hdpi-v4/bass_punch.png
110851        0x1B103         Zip archive data, at least v1.0 to extract, compressed size: 7754, uncompressed size: 7754, name: res/drawable-hdpi-v4/brilliant_treble.png
118682        0x1CF9A         Zip archive data, at least v1.0 to extract, compressed size: 470, uncompressed size: 470, name: res/drawable-hdpi-v4/design_ic_visibility.png
119234        0x1D1C2         Zip archive data, at least v1.0 to extract, compressed size: 507, uncompressed size: 507, name: res/drawable-hdpi-v4/design_ic_visibility_off.png
119827        0x1D413         Zip archive data, at least v1.0 to extract, compressed size: 7877, uncompressed size: 7877, name: res/drawable-hdpi-v4/extreme_bass.png
127777        0x1F321         Zip archive data, at least v1.0 to extract, compressed size: 7480, uncompressed size: 7480, name: res/drawable-hdpi-v4/home_theater.png
135332        0x210A4         Zip archive data, at least v1.0 to extract, compressed size: 6046, uncompressed size: 6046, name: res/drawable-hdpi-v4/ic_launcher.png
141450        0x2288A         Zip archive data, at least v1.0 to extract, compressed size: 7883, uncompressed size: 7883, name: res/drawable-hdpi-v4/live_stage.png
149407        0x2479F         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-hdpi-v4/notification_bg_low_normal.9.png
149708        0x248CC         Zip archive data, at least v1.0 to extract, compressed size: 225, uncompressed size: 225, name: res/drawable-hdpi-v4/notification_bg_low_pressed.9.png
150025        0x24A09         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-hdpi-v4/notification_bg_normal.9.png
150324        0x24B34         Zip archive data, at least v1.0 to extract, compressed size: 225, uncompressed size: 225, name: res/drawable-hdpi-v4/notification_bg_normal_pressed.9.png
150645        0x24C75         Zip archive data, at least v1.0 to extract, compressed size: 93, uncompressed size: 93, name: res/drawable-hdpi-v4/notify_panel_notification_icon_bg.png
150740        0x24CD4         PNG image, 14 x 14, 8-bit colormap, non-interlaced
150833        0x24D31         Zip archive data, at least v1.0 to extract, compressed size: 7410, uncompressed size: 7410, name: res/drawable-hdpi-v4/off.png
158310        0x26A66         Zip archive data, at least v1.0 to extract, compressed size: 220, uncompressed size: 220, name: res/drawable-hdpi-v4/progress_vertical_bg_holo_dark.9.png
158624        0x26BA0         Zip archive data, at least v1.0 to extract, compressed size: 667, uncompressed size: 667, name: res/drawable-hdpi-v4/progress_vertical_primary_holo_dark.9.png
159391        0x26E9F         Zip archive data, at least v1.0 to extract, compressed size: 234, uncompressed size: 234, name: res/drawable-hdpi-v4/progress_vertical_secondary_holo_dark.9.png
159726        0x26FEE         Zip archive data, at least v1.0 to extract, compressed size: 224, uncompressed size: 224, name: res/drawable-hdpi-v4/scrubber_vertical_primary_holo.9.png
160044        0x2712C         Zip archive data, at least v1.0 to extract, compressed size: 224, uncompressed size: 224, name: res/drawable-hdpi-v4/scrubber_vertical_secondary_holo.9.png
160364        0x2726C         Zip archive data, at least v1.0 to extract, compressed size: 217, uncompressed size: 217, name: res/drawable-hdpi-v4/scrubber_vertical_track_holo_dark.9.png
160677        0x273A5         Zip archive data, at least v1.0 to extract, compressed size: 217, uncompressed size: 217, name: res/drawable-hdpi-v4/scrubber_vertical_track_holo_light.9.png
160993        0x274E1         Zip archive data, at least v1.0 to extract, compressed size: 7185, uncompressed size: 7185, name: res/drawable-hdpi-v4/stereo.png
168245        0x29135         Zip archive data, at least v1.0 to extract, compressed size: 7709, uncompressed size: 7709, name: res/drawable-hdpi-v4/vocalizer.png
176025        0x2AF99         Zip archive data, at least v1.0 to extract, compressed size: 199, uncompressed size: 199, name: res/drawable-ldrtl-hdpi-v17/abc_ic_menu_copy_mtrl_am_alpha.png
176323        0x2B0C3         Zip archive data, at least v1.0 to extract, compressed size: 400, uncompressed size: 400, name: res/drawable-ldrtl-hdpi-v17/abc_ic_menu_cut_mtrl_alpha.png
176820        0x2B2B4         Zip archive data, at least v1.0 to extract, compressed size: 367, uncompressed size: 367, name: res/drawable-ldrtl-hdpi-v17/abc_spinner_mtrl_am_alpha.9.png
177283        0x2B483         Zip archive data, at least v1.0 to extract, compressed size: 127, uncompressed size: 127, name: res/drawable-ldrtl-mdpi-v17/abc_ic_menu_copy_mtrl_am_alpha.png
177511        0x2B567         Zip archive data, at least v1.0 to extract, compressed size: 253, uncompressed size: 253, name: res/drawable-ldrtl-mdpi-v17/abc_ic_menu_cut_mtrl_alpha.png
177861        0x2B6C5         Zip archive data, at least v1.0 to extract, compressed size: 342, uncompressed size: 342, name: res/drawable-ldrtl-mdpi-v17/abc_spinner_mtrl_am_alpha.9.png
178298        0x2B87A         Zip archive data, at least v1.0 to extract, compressed size: 178, uncompressed size: 178, name: res/drawable-ldrtl-xhdpi-v17/abc_ic_menu_copy_mtrl_am_alpha.png
178578        0x2B992         Zip archive data, at least v1.0 to extract, compressed size: 494, uncompressed size: 494, name: res/drawable-ldrtl-xhdpi-v17/abc_ic_menu_cut_mtrl_alpha.png
179170        0x2BBE2         Zip archive data, at least v1.0 to extract, compressed size: 483, uncompressed size: 483, name: res/drawable-ldrtl-xhdpi-v17/abc_spinner_mtrl_am_alpha.9.png
179751        0x2BE27         Zip archive data, at least v1.0 to extract, compressed size: 260, uncompressed size: 260, name: res/drawable-ldrtl-xxhdpi-v17/abc_ic_menu_copy_mtrl_am_alpha.png
180112        0x2BF90         Zip archive data, at least v1.0 to extract, compressed size: 705, uncompressed size: 705, name: res/drawable-ldrtl-xxhdpi-v17/abc_ic_menu_cut_mtrl_alpha.png
180913        0x2C2B1         Zip archive data, at least v1.0 to extract, compressed size: 593, uncompressed size: 593, name: res/drawable-ldrtl-xxhdpi-v17/abc_spinner_mtrl_am_alpha.9.png
181605        0x2C565         Zip archive data, at least v1.0 to extract, compressed size: 325, uncompressed size: 325, name: res/drawable-ldrtl-xxxhdpi-v17/abc_ic_menu_copy_mtrl_am_alpha.png
182033        0x2C711         Zip archive data, at least v1.0 to extract, compressed size: 905, uncompressed size: 905, name: res/drawable-ldrtl-xxxhdpi-v17/abc_ic_menu_cut_mtrl_alpha.png
183037        0x2CAFD         Zip archive data, at least v1.0 to extract, compressed size: 518, uncompressed size: 518, name: res/drawable-ldrtl-xxxhdpi-v17/abc_spinner_mtrl_am_alpha.9.png
183654        0x2CD66         Zip archive data, at least v1.0 to extract, compressed size: 274, uncompressed size: 274, name: res/drawable-mdpi-v4/abc_ab_share_pack_mtrl_alpha.9.png
184022        0x2CED6         Zip archive data, at least v1.0 to extract, compressed size: 214, uncompressed size: 214, name: res/drawable-mdpi-v4/abc_btn_check_to_on_mtrl_000.png
184326        0x2D006         Zip archive data, at least v1.0 to extract, compressed size: 321, uncompressed size: 321, name: res/drawable-mdpi-v4/abc_btn_check_to_on_mtrl_015.png
184737        0x2D1A1         Zip archive data, at least v1.0 to extract, compressed size: 324, uncompressed size: 324, name: res/drawable-mdpi-v4/abc_btn_radio_to_on_mtrl_000.png
185152        0x2D340         Zip archive data, at least v1.0 to extract, compressed size: 356, uncompressed size: 356, name: res/drawable-mdpi-v4/abc_btn_radio_to_on_mtrl_015.png
185600        0x2D500         Zip archive data, at least v1.0 to extract, compressed size: 1047, uncompressed size: 1047, name: res/drawable-mdpi-v4/abc_btn_switch_to_on_mtrl_00001.9.png
186743        0x2D977         Zip archive data, at least v1.0 to extract, compressed size: 1124, uncompressed size: 1124, name: res/drawable-mdpi-v4/abc_btn_switch_to_on_mtrl_00012.9.png
187964        0x2DE3C         Zip archive data, at least v1.0 to extract, compressed size: 225, uncompressed size: 225, name: res/drawable-mdpi-v4/abc_cab_background_top_mtrl_alpha.9.png
188285        0x2DF7D         Zip archive data, at least v1.0 to extract, compressed size: 173, uncompressed size: 173, name: res/drawable-mdpi-v4/abc_ic_commit_search_api_mtrl_alpha.png
188557        0x2E08D         Zip archive data, at least v1.0 to extract, compressed size: 133, uncompressed size: 133, name: res/drawable-mdpi-v4/abc_ic_menu_copy_mtrl_am_alpha.png
188781        0x2E16D         Zip archive data, at least v1.0 to extract, compressed size: 251, uncompressed size: 251, name: res/drawable-mdpi-v4/abc_ic_menu_cut_mtrl_alpha.png
189119        0x2E2BF         Zip archive data, at least v1.0 to extract, compressed size: 152, uncompressed size: 152, name: res/drawable-mdpi-v4/abc_ic_menu_paste_mtrl_am_alpha.png
189364        0x2E3B4         Zip archive data, at least v1.0 to extract, compressed size: 139, uncompressed size: 139, name: res/drawable-mdpi-v4/abc_ic_menu_selectall_mtrl_alpha.png
189599        0x2E49F         Zip archive data, at least v1.0 to extract, compressed size: 270, uncompressed size: 270, name: res/drawable-mdpi-v4/abc_ic_menu_share_mtrl_alpha.png
189958        0x2E606         Zip archive data, at least v1.0 to extract, compressed size: 193, uncompressed size: 193, name: res/drawable-mdpi-v4/abc_ic_star_black_16dp.png
190237        0x2E71D         Zip archive data, at least v1.0 to extract, compressed size: 364, uncompressed size: 364, name: res/drawable-mdpi-v4/abc_ic_star_black_36dp.png
190684        0x2E8DC         Zip archive data, at least v1.0 to extract, compressed size: 467, uncompressed size: 467, name: res/drawable-mdpi-v4/abc_ic_star_black_48dp.png
191235        0x2EB03         Zip archive data, at least v1.0 to extract, compressed size: 146, uncompressed size: 146, name: res/drawable-mdpi-v4/abc_ic_star_half_black_16dp.png
191470        0x2EBEE         Zip archive data, at least v1.0 to extract, compressed size: 253, uncompressed size: 253, name: res/drawable-mdpi-v4/abc_ic_star_half_black_36dp.png
191813        0x2ED45         Zip archive data, at least v1.0 to extract, compressed size: 310, uncompressed size: 310, name: res/drawable-mdpi-v4/abc_ic_star_half_black_48dp.png
192214        0x2EED6         Zip archive data, at least v1.0 to extract, compressed size: 167, uncompressed size: 167, name: res/drawable-mdpi-v4/abc_list_divider_mtrl_alpha.9.png
192471        0x2EFD7         Zip archive data, at least v1.0 to extract, compressed size: 222, uncompressed size: 222, name: res/drawable-mdpi-v4/abc_list_focused_holo.9.png
192778        0x2F10A         Zip archive data, at least v1.0 to extract, compressed size: 211, uncompressed size: 211, name: res/drawable-mdpi-v4/abc_list_longpressed_holo.9.png
193079        0x2F237         Zip archive data, at least v1.0 to extract, compressed size: 211, uncompressed size: 211, name: res/drawable-mdpi-v4/abc_list_pressed_holo_dark.9.png
193379        0x2F363         Zip archive data, at least v1.0 to extract, compressed size: 211, uncompressed size: 211, name: res/drawable-mdpi-v4/abc_list_pressed_holo_light.9.png
193683        0x2F493         Zip archive data, at least v1.0 to extract, compressed size: 226, uncompressed size: 226, name: res/drawable-mdpi-v4/abc_list_selector_disabled_holo_dark.9.png
194010        0x2F5DA         Zip archive data, at least v1.0 to extract, compressed size: 227, uncompressed size: 227, name: res/drawable-mdpi-v4/abc_list_selector_disabled_holo_light.9.png
194339        0x2F723         Zip archive data, at least v1.0 to extract, compressed size: 589, uncompressed size: 589, name: res/drawable-mdpi-v4/abc_menu_hardkey_panel_mtrl_mult.9.png
195025        0x2F9D1         Zip archive data, at least v1.0 to extract, compressed size: 850, uncompressed size: 850, name: res/drawable-mdpi-v4/abc_popup_background_mtrl_mult.9.png
195970        0x2FD82         Zip archive data, at least v1.0 to extract, compressed size: 159, uncompressed size: 159, name: res/drawable-mdpi-v4/abc_scrubber_control_off_mtrl_alpha.png
196227        0x2FE83         Zip archive data, at least v1.0 to extract, compressed size: 145, uncompressed size: 145, name: res/drawable-mdpi-v4/abc_scrubber_control_to_pressed_mtrl_000.png
196473        0x2FF79         Zip archive data, at least v1.0 to extract, compressed size: 197, uncompressed size: 197, name: res/drawable-mdpi-v4/abc_scrubber_control_to_pressed_mtrl_005.png
196773        0x300A5         Zip archive data, at least v1.0 to extract, compressed size: 208, uncompressed size: 208, name: res/drawable-mdpi-v4/abc_scrubber_primary_mtrl_alpha.9.png
197076        0x301D4         Zip archive data, at least v1.0 to extract, compressed size: 197, uncompressed size: 197, name: res/drawable-mdpi-v4/abc_scrubber_track_mtrl_alpha.9.png
197365        0x302F5         Zip archive data, at least v1.0 to extract, compressed size: 340, uncompressed size: 340, name: res/drawable-mdpi-v4/abc_spinner_mtrl_am_alpha.9.png
197796        0x304A4         Zip archive data, at least v1.0 to extract, compressed size: 428, uncompressed size: 428, name: res/drawable-mdpi-v4/abc_switch_track_mtrl_alpha.9.png
198316        0x306AC         Zip archive data, at least v1.0 to extract, compressed size: 192, uncompressed size: 192, name: res/drawable-mdpi-v4/abc_tab_indicator_mtrl_alpha.9.png
198600        0x307C8         Zip archive data, at least v1.0 to extract, compressed size: 203, uncompressed size: 203, name: res/drawable-mdpi-v4/abc_text_select_handle_left_mtrl_dark.png
198903        0x308F7         Zip archive data, at least v1.0 to extract, compressed size: 203, uncompressed size: 203, name: res/drawable-mdpi-v4/abc_text_select_handle_left_mtrl_light.png
199207        0x30A27         Zip archive data, at least v1.0 to extract, compressed size: 311, uncompressed size: 311, name: res/drawable-mdpi-v4/abc_text_select_handle_middle_mtrl_dark.png
199619        0x30BC3         Zip archive data, at least v1.0 to extract, compressed size: 310, uncompressed size: 310, name: res/drawable-mdpi-v4/abc_text_select_handle_middle_mtrl_light.png
200030        0x30D5E         Zip archive data, at least v1.0 to extract, compressed size: 187, uncompressed size: 187, name: res/drawable-mdpi-v4/abc_text_select_handle_right_mtrl_dark.png
200319        0x30E7F         Zip archive data, at least v1.0 to extract, compressed size: 186, uncompressed size: 186, name: res/drawable-mdpi-v4/abc_text_select_handle_right_mtrl_light.png
200606        0x30F9E         Zip archive data, at least v1.0 to extract, compressed size: 186, uncompressed size: 186, name: res/drawable-mdpi-v4/abc_textfield_activated_mtrl_alpha.9.png
200890        0x310BA         Zip archive data, at least v1.0 to extract, compressed size: 182, uncompressed size: 182, name: res/drawable-mdpi-v4/abc_textfield_default_mtrl_alpha.9.png
201170        0x311D2         Zip archive data, at least v1.0 to extract, compressed size: 181, uncompressed size: 181, name: res/drawable-mdpi-v4/abc_textfield_search_activated_mtrl_alpha.9.png
201457        0x312F1         Zip archive data, at least v1.0 to extract, compressed size: 180, uncompressed size: 180, name: res/drawable-mdpi-v4/abc_textfield_search_default_mtrl_alpha.9.png
201740        0x3140C         Zip archive data, at least v1.0 to extract, compressed size: 3839, uncompressed size: 3839, name: res/drawable-mdpi-v4/balanced.png
205651        0x32353         Zip archive data, at least v1.0 to extract, compressed size: 2484, uncompressed size: 2484, name: res/drawable-mdpi-v4/bass_punch.png
208208        0x32D50         Zip archive data, at least v1.0 to extract, compressed size: 4388, uncompressed size: 4388, name: res/drawable-mdpi-v4/brilliant_treble.png
212676        0x33EC4         Zip archive data, at least v1.0 to extract, compressed size: 309, uncompressed size: 309, name: res/drawable-mdpi-v4/design_ic_visibility.png
213069        0x3404D         Zip archive data, at least v1.0 to extract, compressed size: 351, uncompressed size: 351, name: res/drawable-mdpi-v4/design_ic_visibility_off.png
213507        0x34203         Zip archive data, at least v1.0 to extract, compressed size: 4581, uncompressed size: 4581, name: res/drawable-mdpi-v4/extreme_bass.png
218161        0x35431         Zip archive data, at least v1.0 to extract, compressed size: 3958, uncompressed size: 3958, name: res/drawable-mdpi-v4/home_theater.png
222194        0x363F2         Zip archive data, at least v1.0 to extract, compressed size: 3571, uncompressed size: 3571, name: res/drawable-mdpi-v4/ic_launcher.png
225839        0x3722F         Zip archive data, at least v1.0 to extract, compressed size: 4576, uncompressed size: 4576, name: res/drawable-mdpi-v4/live_stage.png
230488        0x38458         Zip archive data, at least v1.0 to extract, compressed size: 215, uncompressed size: 215, name: res/drawable-mdpi-v4/notification_bg_low_normal.9.png
230795        0x3858B         Zip archive data, at least v1.0 to extract, compressed size: 223, uncompressed size: 223, name: res/drawable-mdpi-v4/notification_bg_low_pressed.9.png
231111        0x386C7         Zip archive data, at least v1.0 to extract, compressed size: 215, uncompressed size: 215, name: res/drawable-mdpi-v4/notification_bg_normal.9.png
231411        0x387F3         Zip archive data, at least v1.0 to extract, compressed size: 223, uncompressed size: 223, name: res/drawable-mdpi-v4/notification_bg_normal_pressed.9.png
231727        0x3892F         Zip archive data, at least v1.0 to extract, compressed size: 93, uncompressed size: 93, name: res/drawable-mdpi-v4/notify_panel_notification_icon_bg.png
231824        0x38990         PNG image, 15 x 15, 8-bit colormap, non-interlaced
231917        0x389ED         Zip archive data, at least v1.0 to extract, compressed size: 3918, uncompressed size: 3918, name: res/drawable-mdpi-v4/off.png
235902        0x3997E         Zip archive data, at least v1.0 to extract, compressed size: 216, uncompressed size: 216, name: res/drawable-mdpi-v4/progress_vertical_bg_holo_dark.9.png
236212        0x39AB4         Zip archive data, at least v1.0 to extract, compressed size: 484, uncompressed size: 484, name: res/drawable-mdpi-v4/progress_vertical_primary_holo_dark.9.png
236796        0x39CFC         Zip archive data, at least v1.0 to extract, compressed size: 227, uncompressed size: 227, name: res/drawable-mdpi-v4/progress_vertical_secondary_holo_dark.9.png
237123        0x39E43         Zip archive data, at least v1.0 to extract, compressed size: 217, uncompressed size: 217, name: res/drawable-mdpi-v4/scrubber_vertical_primary_holo.9.png
237433        0x39F79         Zip archive data, at least v1.0 to extract, compressed size: 217, uncompressed size: 217, name: res/drawable-mdpi-v4/scrubber_vertical_secondary_holo.9.png
237745        0x3A0B1         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-mdpi-v4/scrubber_vertical_track_holo_dark.9.png
238056        0x3A1E8         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-mdpi-v4/scrubber_vertical_track_holo_light.9.png
238368        0x3A320         Zip archive data, at least v1.0 to extract, compressed size: 2484, uncompressed size: 2484, name: res/drawable-mdpi-v4/stereo.png
240920        0x3AD18         Zip archive data, at least v1.0 to extract, compressed size: 4336, uncompressed size: 4336, name: res/drawable-mdpi-v4/vocalizer.png
245328        0x3BE50         Zip archive data, at least v1.0 to extract, compressed size: 297, uncompressed size: 297, name: res/drawable-xhdpi-v4/abc_ab_share_pack_mtrl_alpha.9.png
245717        0x3BFD5         Zip archive data, at least v1.0 to extract, compressed size: 281, uncompressed size: 281, name: res/drawable-xhdpi-v4/abc_btn_check_to_on_mtrl_000.png
246089        0x3C149         Zip archive data, at least v1.0 to extract, compressed size: 432, uncompressed size: 432, name: res/drawable-xhdpi-v4/abc_btn_check_to_on_mtrl_015.png
246612        0x3C354         Zip archive data, at least v1.0 to extract, compressed size: 651, uncompressed size: 651, name: res/drawable-xhdpi-v4/abc_btn_radio_to_on_mtrl_000.png
247355        0x3C63B         Zip archive data, at least v1.0 to extract, compressed size: 785, uncompressed size: 785, name: res/drawable-xhdpi-v4/abc_btn_radio_to_on_mtrl_015.png
248233        0x3C9A9         Zip archive data, at least v1.0 to extract, compressed size: 2259, uncompressed size: 2259, name: res/drawable-xhdpi-v4/abc_btn_switch_to_on_mtrl_00001.9.png
250587        0x3D2DB         Zip archive data, at least v1.0 to extract, compressed size: 2606, uncompressed size: 2606, name: res/drawable-xhdpi-v4/abc_btn_switch_to_on_mtrl_00012.9.png
253290        0x3DD6A         Zip archive data, at least v1.0 to extract, compressed size: 234, uncompressed size: 234, name: res/drawable-xhdpi-v4/abc_cab_background_top_mtrl_alpha.9.png
253622        0x3DEB6         Zip archive data, at least v1.0 to extract, compressed size: 228, uncompressed size: 228, name: res/drawable-xhdpi-v4/abc_ic_commit_search_api_mtrl_alpha.png
253948        0x3DFFC         Zip archive data, at least v1.0 to extract, compressed size: 178, uncompressed size: 178, name: res/drawable-xhdpi-v4/abc_ic_menu_copy_mtrl_am_alpha.png
254218        0x3E10A         Zip archive data, at least v1.0 to extract, compressed size: 492, uncompressed size: 492, name: res/drawable-xhdpi-v4/abc_ic_menu_cut_mtrl_alpha.png
254800        0x3E350         Zip archive data, at least v1.0 to extract, compressed size: 243, uncompressed size: 243, name: res/drawable-xhdpi-v4/abc_ic_menu_paste_mtrl_am_alpha.png
255139        0x3E4A3         Zip archive data, at least v1.0 to extract, compressed size: 183, uncompressed size: 183, name: res/drawable-xhdpi-v4/abc_ic_menu_selectall_mtrl_alpha.png
255419        0x3E5BB         Zip archive data, at least v1.0 to extract, compressed size: 480, uncompressed size: 480, name: res/drawable-xhdpi-v4/abc_ic_menu_share_mtrl_alpha.png
255992        0x3E7F8         Zip archive data, at least v1.0 to extract, compressed size: 333, uncompressed size: 333, name: res/drawable-xhdpi-v4/abc_ic_star_black_16dp.png
256409        0x3E999         Zip archive data, at least v1.0 to extract, compressed size: 652, uncompressed size: 652, name: res/drawable-xhdpi-v4/abc_ic_star_black_36dp.png
257148        0x3EC7C         Zip archive data, at least v1.0 to extract, compressed size: 887, uncompressed size: 887, name: res/drawable-xhdpi-v4/abc_ic_star_black_48dp.png
258119        0x3F047         Zip archive data, at least v1.0 to extract, compressed size: 235, uncompressed size: 235, name: res/drawable-xhdpi-v4/abc_ic_star_half_black_16dp.png
258443        0x3F18B         Zip archive data, at least v1.0 to extract, compressed size: 421, uncompressed size: 421, name: res/drawable-xhdpi-v4/abc_ic_star_half_black_36dp.png
258953        0x3F389         Zip archive data, at least v1.0 to extract, compressed size: 548, uncompressed size: 548, name: res/drawable-xhdpi-v4/abc_ic_star_half_black_48dp.png
259592        0x3F608         Zip archive data, at least v1.0 to extract, compressed size: 167, uncompressed size: 167, name: res/drawable-xhdpi-v4/abc_list_divider_mtrl_alpha.9.png
259851        0x3F70B         Zip archive data, at least v1.0 to extract, compressed size: 244, uncompressed size: 244, name: res/drawable-xhdpi-v4/abc_list_focused_holo.9.png
260180        0x3F854         Zip archive data, at least v1.0 to extract, compressed size: 214, uncompressed size: 214, name: res/drawable-xhdpi-v4/abc_list_longpressed_holo.9.png
260486        0x3F986         Zip archive data, at least v1.0 to extract, compressed size: 214, uncompressed size: 214, name: res/drawable-xhdpi-v4/abc_list_pressed_holo_dark.9.png
260790        0x3FAB6         Zip archive data, at least v1.0 to extract, compressed size: 214, uncompressed size: 214, name: res/drawable-xhdpi-v4/abc_list_pressed_holo_light.9.png
261098        0x3FBEA         Zip archive data, at least v1.0 to extract, compressed size: 254, uncompressed size: 254, name: res/drawable-xhdpi-v4/abc_list_selector_disabled_holo_dark.9.png
261454        0x3FD4E         Zip archive data, at least v1.0 to extract, compressed size: 253, uncompressed size: 253, name: res/drawable-xhdpi-v4/abc_list_selector_disabled_holo_light.9.png
261809        0x3FEB1         Zip archive data, at least v1.0 to extract, compressed size: 1122, uncompressed size: 1122, name: res/drawable-xhdpi-v4/abc_menu_hardkey_panel_mtrl_mult.9.png
263030        0x40376         Zip archive data, at least v1.0 to extract, compressed size: 1785, uncompressed size: 1785, name: res/drawable-xhdpi-v4/abc_popup_background_mtrl_mult.9.png
264909        0x40ACD         Zip archive data, at least v1.0 to extract, compressed size: 267, uncompressed size: 267, name: res/drawable-xhdpi-v4/abc_scrubber_control_off_mtrl_alpha.png
265275        0x40C3B         Zip archive data, at least v1.0 to extract, compressed size: 267, uncompressed size: 267, name: res/drawable-xhdpi-v4/abc_scrubber_control_to_pressed_mtrl_000.png
265647        0x40DAF         Zip archive data, at least v1.0 to extract, compressed size: 391, uncompressed size: 391, name: res/drawable-xhdpi-v4/abc_scrubber_control_to_pressed_mtrl_005.png
266143        0x40F9F         Zip archive data, at least v1.0 to extract, compressed size: 219, uncompressed size: 219, name: res/drawable-xhdpi-v4/abc_scrubber_primary_mtrl_alpha.9.png
266459        0x410DB         Zip archive data, at least v1.0 to extract, compressed size: 207, uncompressed size: 207, name: res/drawable-xhdpi-v4/abc_scrubber_track_mtrl_alpha.9.png
266759        0x41207         Zip archive data, at least v1.0 to extract, compressed size: 489, uncompressed size: 489, name: res/drawable-xhdpi-v4/abc_spinner_mtrl_am_alpha.9.png
267337        0x41449         Zip archive data, at least v1.0 to extract, compressed size: 741, uncompressed size: 741, name: res/drawable-xhdpi-v4/abc_switch_track_mtrl_alpha.9.png
268169        0x41789         Zip archive data, at least v1.0 to extract, compressed size: 205, uncompressed size: 205, name: res/drawable-xhdpi-v4/abc_tab_indicator_mtrl_alpha.9.png
268469        0x418B5         Zip archive data, at least v1.0 to extract, compressed size: 336, uncompressed size: 336, name: res/drawable-xhdpi-v4/abc_text_select_handle_left_mtrl_dark.png
268904        0x41A68         Zip archive data, at least v1.0 to extract, compressed size: 335, uncompressed size: 335, name: res/drawable-xhdpi-v4/abc_text_select_handle_left_mtrl_light.png
269339        0x41C1B         Zip archive data, at least v1.0 to extract, compressed size: 583, uncompressed size: 583, name: res/drawable-xhdpi-v4/abc_text_select_handle_middle_mtrl_dark.png
270023        0x41EC7         Zip archive data, at least v1.0 to extract, compressed size: 585, uncompressed size: 585, name: res/drawable-xhdpi-v4/abc_text_select_handle_middle_mtrl_light.png
270713        0x42179         Zip archive data, at least v1.0 to extract, compressed size: 319, uncompressed size: 319, name: res/drawable-xhdpi-v4/abc_text_select_handle_right_mtrl_dark.png
271135        0x4231F         Zip archive data, at least v1.0 to extract, compressed size: 318, uncompressed size: 318, name: res/drawable-xhdpi-v4/abc_text_select_handle_right_mtrl_light.png
271554        0x424C2         Zip archive data, at least v1.0 to extract, compressed size: 198, uncompressed size: 198, name: res/drawable-xhdpi-v4/abc_textfield_activated_mtrl_alpha.9.png
271850        0x425EA         Zip archive data, at least v1.0 to extract, compressed size: 197, uncompressed size: 197, name: res/drawable-xhdpi-v4/abc_textfield_default_mtrl_alpha.9.png
272145        0x42711         Zip archive data, at least v1.0 to extract, compressed size: 190, uncompressed size: 190, name: res/drawable-xhdpi-v4/abc_textfield_search_activated_mtrl_alpha.9.png
272442        0x4283A         Zip archive data, at least v1.0 to extract, compressed size: 190, uncompressed size: 190, name: res/drawable-xhdpi-v4/abc_textfield_search_default_mtrl_alpha.9.png
272738        0x42962         Zip archive data, at least v1.0 to extract, compressed size: 7816, uncompressed size: 7816, name: res/drawable-xhdpi-v4/balanced.png
280624        0x44830         Zip archive data, at least v1.0 to extract, compressed size: 6645, uncompressed size: 6645, name: res/drawable-xhdpi-v4/bass_punch.png
287341        0x4626D         Zip archive data, at least v1.0 to extract, compressed size: 8593, uncompressed size: 8593, name: res/drawable-xhdpi-v4/brilliant_treble.png
296013        0x4844D         Zip archive data, at least v1.0 to extract, compressed size: 593, uncompressed size: 593, name: res/drawable-xhdpi-v4/design_ic_visibility.png
296689        0x486F1         Zip archive data, at least v1.0 to extract, compressed size: 629, uncompressed size: 629, name: res/drawable-xhdpi-v4/design_ic_visibility_off.png
297405        0x489BD         Zip archive data, at least v1.0 to extract, compressed size: 8829, uncompressed size: 8829, name: res/drawable-xhdpi-v4/extreme_bass.png
306309        0x4AC85         Zip archive data, at least v1.0 to extract, compressed size: 8165, uncompressed size: 8165, name: res/drawable-xhdpi-v4/home_theater.png
314549        0x4CCB5         Zip archive data, at least v1.0 to extract, compressed size: 94, uncompressed size: 94, name: res/drawable-xhdpi-v4/ic_equalizer.png
314718        0x4CD5E         Zip archive data, at least v1.0 to extract, compressed size: 9388, uncompressed size: 9388, name: res/drawable-xhdpi-v4/ic_launcher.png
324180        0x4F254         Zip archive data, at least v1.0 to extract, compressed size: 8829, uncompressed size: 8829, name: res/drawable-xhdpi-v4/live_stage.png
333081        0x51519         Zip archive data, at least v1.0 to extract, compressed size: 221, uncompressed size: 221, name: res/drawable-xhdpi-v4/notification_bg_low_normal.9.png
333393        0x51651         Zip archive data, at least v1.0 to extract, compressed size: 252, uncompressed size: 252, name: res/drawable-xhdpi-v4/notification_bg_low_pressed.9.png
333736        0x517A8         Zip archive data, at least v1.0 to extract, compressed size: 221, uncompressed size: 221, name: res/drawable-xhdpi-v4/notification_bg_normal.9.png
334045        0x518DD         Zip archive data, at least v1.0 to extract, compressed size: 247, uncompressed size: 247, name: res/drawable-xhdpi-v4/notification_bg_normal_pressed.9.png
334387        0x51A33         Zip archive data, at least v1.0 to extract, compressed size: 99, uncompressed size: 99, name: res/drawable-xhdpi-v4/notify_panel_notification_icon_bg.png
334583        0x51AF7         Zip archive data, at least v1.0 to extract, compressed size: 8137, uncompressed size: 8137, name: res/drawable-xhdpi-v4/off.png
342785        0x53B01         Zip archive data, at least v1.0 to extract, compressed size: 889, uncompressed size: 889, name: res/drawable-xhdpi-v4/progress_vertical_primary_holo_dark.9.png
343773        0x53EDD         Zip archive data, at least v1.0 to extract, compressed size: 230, uncompressed size: 230, name: res/drawable-xhdpi-v4/progress_vertical_secondary_holo_dark.9.png
344106        0x5402A         Zip archive data, at least v1.0 to extract, compressed size: 230, uncompressed size: 230, name: res/drawable-xhdpi-v4/scrubber_vertical_primary_holo.9.png
344430        0x5416E         Zip archive data, at least v1.0 to extract, compressed size: 230, uncompressed size: 230, name: res/drawable-xhdpi-v4/scrubber_vertical_secondary_holo.9.png
344758        0x542B6         Zip archive data, at least v1.0 to extract, compressed size: 228, uncompressed size: 228, name: res/drawable-xhdpi-v4/scrubber_vertical_track_holo_dark.9.png
345084        0x543FC         Zip archive data, at least v1.0 to extract, compressed size: 228, uncompressed size: 228, name: res/drawable-xhdpi-v4/scrubber_vertical_track_holo_light.9.png
345412        0x54544         Zip archive data, at least v1.0 to extract, compressed size: 6645, uncompressed size: 6645, name: res/drawable-xhdpi-v4/stereo.png
352125        0x55F7D         Zip archive data, at least v1.0 to extract, compressed size: 8668, uncompressed size: 8668, name: res/drawable-xhdpi-v4/vocalizer.png
360864        0x581A0         Zip archive data, at least v1.0 to extract, compressed size: 305, uncompressed size: 305, name: res/drawable-xxhdpi-v4/abc_ab_share_pack_mtrl_alpha.9.png
361265        0x58331         Zip archive data, at least v1.0 to extract, compressed size: 307, uncompressed size: 307, name: res/drawable-xxhdpi-v4/abc_btn_check_to_on_mtrl_000.png
361663        0x584BF         Zip archive data, at least v1.0 to extract, compressed size: 593, uncompressed size: 593, name: res/drawable-xxhdpi-v4/abc_btn_check_to_on_mtrl_015.png
362349        0x5876D         Zip archive data, at least v1.0 to extract, compressed size: 984, uncompressed size: 984, name: res/drawable-xxhdpi-v4/abc_btn_radio_to_on_mtrl_000.png
363424        0x58BA0         Zip archive data, at least v1.0 to extract, compressed size: 1208, uncompressed size: 1208, name: res/drawable-xxhdpi-v4/abc_btn_radio_to_on_mtrl_015.png
364724        0x590B4         Zip archive data, at least v1.0 to extract, compressed size: 3755, uncompressed size: 3755, name: res/drawable-xxhdpi-v4/abc_btn_switch_to_on_mtrl_00001.9.png
368575        0x59FBF         Zip archive data, at least v1.0 to extract, compressed size: 2804, uncompressed size: 2804, name: res/drawable-xxhdpi-v4/abc_btn_switch_to_on_mtrl_00012.9.png
371476        0x5AB14         Zip archive data, at least v1.0 to extract, compressed size: 246, uncompressed size: 246, name: res/drawable-xxhdpi-v4/abc_cab_background_top_mtrl_alpha.9.png
371749        0x5AC25         Zlib compressed data, best compression
371822        0x5AC6E         Zip archive data, at least v1.0 to extract, compressed size: 224, uncompressed size: 224, name: res/drawable-xxhdpi-v4/abc_ic_commit_search_api_mtrl_alpha.png
372144        0x5ADB0         Zip archive data, at least v1.0 to extract, compressed size: 263, uncompressed size: 263, name: res/drawable-xxhdpi-v4/abc_ic_menu_copy_mtrl_am_alpha.png
372503        0x5AF17         Zip archive data, at least v1.0 to extract, compressed size: 710, uncompressed size: 710, name: res/drawable-xxhdpi-v4/abc_ic_menu_cut_mtrl_alpha.png
373302        0x5B236         Zip archive data, at least v1.0 to extract, compressed size: 348, uncompressed size: 348, name: res/drawable-xxhdpi-v4/abc_ic_menu_paste_mtrl_am_alpha.png
373744        0x5B3F0         Zip archive data, at least v1.0 to extract, compressed size: 262, uncompressed size: 262, name: res/drawable-xxhdpi-v4/abc_ic_menu_selectall_mtrl_alpha.png
374102        0x5B556         Zip archive data, at least v1.0 to extract, compressed size: 700, uncompressed size: 700, name: res/drawable-xxhdpi-v4/abc_ic_menu_share_mtrl_alpha.png
374896        0x5B870         Zip archive data, at least v1.0 to extract, compressed size: 459, uncompressed size: 459, name: res/drawable-xxhdpi-v4/abc_ic_star_black_16dp.png
375443        0x5BA93         Zip archive data, at least v1.0 to extract, compressed size: 983, uncompressed size: 983, name: res/drawable-xxhdpi-v4/abc_ic_star_black_36dp.png
376511        0x5BEBF         Zip archive data, at least v1.0 to extract, compressed size: 1291, uncompressed size: 1291, name: res/drawable-xxhdpi-v4/abc_ic_star_black_48dp.png
377887        0x5C41F         Zip archive data, at least v1.0 to extract, compressed size: 309, uncompressed size: 309, name: res/drawable-xxhdpi-v4/abc_ic_star_half_black_16dp.png
378289        0x5C5B1         Zip archive data, at least v1.0 to extract, compressed size: 577, uncompressed size: 577, name: res/drawable-xxhdpi-v4/abc_ic_star_half_black_36dp.png
378957        0x5C84D         Zip archive data, at least v1.0 to extract, compressed size: 789, uncompressed size: 789, name: res/drawable-xxhdpi-v4/abc_ic_star_half_black_48dp.png
379837        0x5CBBD         Zip archive data, at least v1.0 to extract, compressed size: 171, uncompressed size: 171, name: res/drawable-xxhdpi-v4/abc_list_divider_mtrl_alpha.9.png
380103        0x5CCC7         Zip archive data, at least v1.0 to extract, compressed size: 245, uncompressed size: 245, name: res/drawable-xxhdpi-v4/abc_list_focused_holo.9.png
380437        0x5CE15         Zip archive data, at least v1.0 to extract, compressed size: 221, uncompressed size: 221, name: res/drawable-xxhdpi-v4/abc_list_longpressed_holo.9.png
380749        0x5CF4D         Zip archive data, at least v1.0 to extract, compressed size: 221, uncompressed size: 221, name: res/drawable-xxhdpi-v4/abc_list_pressed_holo_dark.9.png
381061        0x5D085         Zip archive data, at least v1.0 to extract, compressed size: 221, uncompressed size: 221, name: res/drawable-xxhdpi-v4/abc_list_pressed_holo_light.9.png
381377        0x5D1C1         Zip archive data, at least v1.0 to extract, compressed size: 307, uncompressed size: 307, name: res/drawable-xxhdpi-v4/abc_list_selector_disabled_holo_dark.9.png
381787        0x5D35B         Zip archive data, at least v1.0 to extract, compressed size: 305, uncompressed size: 305, name: res/drawable-xxhdpi-v4/abc_list_selector_disabled_holo_light.9.png
382197        0x5D4F5         Zip archive data, at least v1.0 to extract, compressed size: 1779, uncompressed size: 1779, name: res/drawable-xxhdpi-v4/abc_menu_hardkey_panel_mtrl_mult.9.png
384075        0x5DC4B         Zip archive data, at least v1.0 to extract, compressed size: 2774, uncompressed size: 2774, name: res/drawable-xxhdpi-v4/abc_popup_background_mtrl_mult.9.png
386946        0x5E782         Zip archive data, at least v1.0 to extract, compressed size: 322, uncompressed size: 322, name: res/drawable-xxhdpi-v4/abc_scrubber_control_off_mtrl_alpha.png
387366        0x5E926         Zip archive data, at least v1.0 to extract, compressed size: 403, uncompressed size: 403, name: res/drawable-xxhdpi-v4/abc_scrubber_control_to_pressed_mtrl_000.png
387875        0x5EB23         Zip archive data, at least v1.0 to extract, compressed size: 595, uncompressed size: 595, name: res/drawable-xxhdpi-v4/abc_scrubber_control_to_pressed_mtrl_005.png
388575        0x5EDDF         Zip archive data, at least v1.0 to extract, compressed size: 218, uncompressed size: 218, name: res/drawable-xxhdpi-v4/abc_scrubber_primary_mtrl_alpha.9.png
388890        0x5EF1A         Zip archive data, at least v1.0 to extract, compressed size: 212, uncompressed size: 212, name: res/drawable-xxhdpi-v4/abc_scrubber_track_mtrl_alpha.9.png
389196        0x5F04C         Zip archive data, at least v1.0 to extract, compressed size: 595, uncompressed size: 595, name: res/drawable-xxhdpi-v4/abc_spinner_mtrl_am_alpha.9.png
389883        0x5F2FB         Zip archive data, at least v1.0 to extract, compressed size: 1060, uncompressed size: 1060, name: res/drawable-xxhdpi-v4/abc_switch_track_mtrl_alpha.9.png
391036        0x5F77C         Zip archive data, at least v1.0 to extract, compressed size: 210, uncompressed size: 210, name: res/drawable-xxhdpi-v4/abc_tab_indicator_mtrl_alpha.9.png
391342        0x5F8AE         Zip archive data, at least v1.0 to extract, compressed size: 420, uncompressed size: 420, name: res/drawable-xxhdpi-v4/abc_text_select_handle_left_mtrl_dark.png
391864        0x5FAB8         Zip archive data, at least v1.0 to extract, compressed size: 420, uncompressed size: 420, name: res/drawable-xxhdpi-v4/abc_text_select_handle_left_mtrl_light.png
392388        0x5FCC4         Zip archive data, at least v1.0 to extract, compressed size: 752, uncompressed size: 752, name: res/drawable-xxhdpi-v4/abc_text_select_handle_middle_mtrl_dark.png
393244        0x6001C         Zip archive data, at least v1.0 to extract, compressed size: 753, uncompressed size: 753, name: res/drawable-xxhdpi-v4/abc_text_select_handle_middle_mtrl_light.png
394101        0x60375         Zip archive data, at least v1.0 to extract, compressed size: 422, uncompressed size: 422, name: res/drawable-xxhdpi-v4/abc_text_select_handle_right_mtrl_dark.png
394626        0x60582         Zip archive data, at least v1.0 to extract, compressed size: 422, uncompressed size: 422, name: res/drawable-xxhdpi-v4/abc_text_select_handle_right_mtrl_light.png
395150        0x6078E         Zip archive data, at least v1.0 to extract, compressed size: 202, uncompressed size: 202, name: res/drawable-xxhdpi-v4/abc_textfield_activated_mtrl_alpha.9.png
395454        0x608BE         Zip archive data, at least v1.0 to extract, compressed size: 204, uncompressed size: 204, name: res/drawable-xxhdpi-v4/abc_textfield_default_mtrl_alpha.9.png
395756        0x609EC         Zip archive data, at least v1.0 to extract, compressed size: 193, uncompressed size: 193, name: res/drawable-xxhdpi-v4/abc_textfield_search_activated_mtrl_alpha.9.png
396057        0x60B19         Zip archive data, at least v1.0 to extract, compressed size: 196, uncompressed size: 196, name: res/drawable-xxhdpi-v4/abc_textfield_search_default_mtrl_alpha.9.png
396360        0x60C48         Zip archive data, at least v1.0 to extract, compressed size: 11928, uncompressed size: 11928, name: res/drawable-xxhdpi-v4/balanced.png
408360        0x63B28         Zip archive data, at least v1.0 to extract, compressed size: 10344, uncompressed size: 10344, name: res/drawable-xxhdpi-v4/bass_punch.png
418780        0x663DC         Zip archive data, at least v1.0 to extract, compressed size: 13143, uncompressed size: 13143, name: res/drawable-xxhdpi-v4/brilliant_treble.png
432003        0x69783         Zip archive data, at least v1.0 to extract, compressed size: 868, uncompressed size: 868, name: res/drawable-xxhdpi-v4/design_ic_visibility.png
432956        0x69B3C         Zip archive data, at least v1.0 to extract, compressed size: 884, uncompressed size: 884, name: res/drawable-xxhdpi-v4/design_ic_visibility_off.png
433928        0x69F08         Zip archive data, at least v1.0 to extract, compressed size: 13402, uncompressed size: 13402, name: res/drawable-xxhdpi-v4/extreme_bass.png
447406        0x6D3AE         Zip archive data, at least v1.0 to extract, compressed size: 12493, uncompressed size: 12493, name: res/drawable-xxhdpi-v4/home_theater.png
459977        0x704C9         Zip archive data, at least v1.0 to extract, compressed size: 16504, uncompressed size: 16504, name: res/drawable-xxhdpi-v4/ic_launcher.png
476556        0x7458C         Zip archive data, at least v1.0 to extract, compressed size: 13383, uncompressed size: 13383, name: res/drawable-xxhdpi-v4/live_stage.png
490015        0x77A1F         Zip archive data, at least v1.0 to extract, compressed size: 12518, uncompressed size: 12518, name: res/drawable-xxhdpi-v4/off.png
502602        0x7AB4A         Zip archive data, at least v1.0 to extract, compressed size: 10344, uncompressed size: 10344, name: res/drawable-xxhdpi-v4/stereo.png
513016        0x7D3F8         Zip archive data, at least v1.0 to extract, compressed size: 13352, uncompressed size: 13352, name: res/drawable-xxhdpi-v4/vocalizer.png
526440        0x80868         Zip archive data, at least v1.0 to extract, compressed size: 275, uncompressed size: 275, name: res/drawable-xxxhdpi-v4/abc_btn_check_to_on_mtrl_000.png
526807        0x809D7         Zip archive data, at least v1.0 to extract, compressed size: 476, uncompressed size: 476, name: res/drawable-xxxhdpi-v4/abc_btn_check_to_on_mtrl_015.png
527376        0x80C10         Zip archive data, at least v1.0 to extract, compressed size: 785, uncompressed size: 785, name: res/drawable-xxxhdpi-v4/abc_btn_radio_to_on_mtrl_000.png
528253        0x80F7D         Zip archive data, at least v1.0 to extract, compressed size: 946, uncompressed size: 946, name: res/drawable-xxxhdpi-v4/abc_btn_radio_to_on_mtrl_015.png
529294        0x8138E         Zip archive data, at least v1.0 to extract, compressed size: 3524, uncompressed size: 3524, name: res/drawable-xxxhdpi-v4/abc_btn_switch_to_on_mtrl_00001.9.png
532916        0x821B4         Zip archive data, at least v1.0 to extract, compressed size: 3853, uncompressed size: 3853, name: res/drawable-xxxhdpi-v4/abc_btn_switch_to_on_mtrl_00012.9.png
536869        0x83125         Zip archive data, at least v1.0 to extract, compressed size: 327, uncompressed size: 327, name: res/drawable-xxxhdpi-v4/abc_ic_menu_copy_mtrl_am_alpha.png
537291        0x832CB         Zip archive data, at least v1.0 to extract, compressed size: 910, uncompressed size: 910, name: res/drawable-xxxhdpi-v4/abc_ic_menu_cut_mtrl_alpha.png
538294        0x836B6         Zip archive data, at least v1.0 to extract, compressed size: 461, uncompressed size: 461, name: res/drawable-xxxhdpi-v4/abc_ic_menu_paste_mtrl_am_alpha.png
538853        0x838E5         Zip archive data, at least v1.0 to extract, compressed size: 305, uncompressed size: 305, name: res/drawable-xxxhdpi-v4/abc_ic_menu_selectall_mtrl_alpha.png
539257        0x83A79         Zip archive data, at least v1.0 to extract, compressed size: 899, uncompressed size: 899, name: res/drawable-xxxhdpi-v4/abc_ic_menu_share_mtrl_alpha.png
540251        0x83E5B         Zip archive data, at least v1.0 to extract, compressed size: 599, uncompressed size: 599, name: res/drawable-xxxhdpi-v4/abc_ic_star_black_16dp.png
540939        0x8410B         Zip archive data, at least v1.0 to extract, compressed size: 1269, uncompressed size: 1269, name: res/drawable-xxxhdpi-v4/abc_ic_star_black_36dp.png
542297        0x84659         Zip archive data, at least v1.0 to extract, compressed size: 1680, uncompressed size: 1680, name: res/drawable-xxxhdpi-v4/abc_ic_star_black_48dp.png
544064        0x84D40         Zip archive data, at least v1.0 to extract, compressed size: 376, uncompressed size: 376, name: res/drawable-xxxhdpi-v4/abc_ic_star_half_black_16dp.png
544532        0x84F14         Zip archive data, at least v1.0 to extract, compressed size: 760, uncompressed size: 760, name: res/drawable-xxxhdpi-v4/abc_ic_star_half_black_36dp.png
545384        0x85268         Zip archive data, at least v1.0 to extract, compressed size: 991, uncompressed size: 991, name: res/drawable-xxxhdpi-v4/abc_ic_star_half_black_48dp.png
546467        0x856A3         Zip archive data, at least v1.0 to extract, compressed size: 415, uncompressed size: 415, name: res/drawable-xxxhdpi-v4/abc_scrubber_control_to_pressed_mtrl_000.png
546987        0x858AB         Zip archive data, at least v1.0 to extract, compressed size: 631, uncompressed size: 631, name: res/drawable-xxxhdpi-v4/abc_scrubber_control_to_pressed_mtrl_005.png
547723        0x85B8B         Zip archive data, at least v1.0 to extract, compressed size: 513, uncompressed size: 513, name: res/drawable-xxxhdpi-v4/abc_spinner_mtrl_am_alpha.9.png
548329        0x85DE9         Zip archive data, at least v1.0 to extract, compressed size: 1025, uncompressed size: 1025, name: res/drawable-xxxhdpi-v4/abc_switch_track_mtrl_alpha.9.png
549449        0x86249         Zip archive data, at least v1.0 to extract, compressed size: 208, uncompressed size: 208, name: res/drawable-xxxhdpi-v4/abc_tab_indicator_mtrl_alpha.9.png
549752        0x86378         Zip archive data, at least v1.0 to extract, compressed size: 513, uncompressed size: 513, name: res/drawable-xxxhdpi-v4/abc_text_select_handle_left_mtrl_dark.png
550369        0x865E1         Zip archive data, at least v1.0 to extract, compressed size: 513, uncompressed size: 513, name: res/drawable-xxxhdpi-v4/abc_text_select_handle_left_mtrl_light.png
550985        0x86849         Zip archive data, at least v1.0 to extract, compressed size: 513, uncompressed size: 513, name: res/drawable-xxxhdpi-v4/abc_text_select_handle_right_mtrl_dark.png
551601        0x86AB1         Zip archive data, at least v1.0 to extract, compressed size: 513, uncompressed size: 513, name: res/drawable-xxxhdpi-v4/abc_text_select_handle_right_mtrl_light.png
552217        0x86D19         Zip archive data, at least v1.0 to extract, compressed size: 17094, uncompressed size: 17094, name: res/drawable-xxxhdpi-v4/balanced.png
569386        0x8B02A         Zip archive data, at least v1.0 to extract, compressed size: 15195, uncompressed size: 15195, name: res/drawable-xxxhdpi-v4/bass_punch.png
584655        0x8EBCF         Zip archive data, at least v1.0 to extract, compressed size: 18797, uncompressed size: 18797, name: res/drawable-xxxhdpi-v4/brilliant_treble.png
603533        0x9358D         Zip archive data, at least v1.0 to extract, compressed size: 1155, uncompressed size: 1155, name: res/drawable-xxxhdpi-v4/design_ic_visibility.png
604775        0x93A67         Zip archive data, at least v1.0 to extract, compressed size: 1201, uncompressed size: 1201, name: res/drawable-xxxhdpi-v4/design_ic_visibility_off.png
606065        0x93F71         Zip archive data, at least v1.0 to extract, compressed size: 18923, uncompressed size: 18923, name: res/drawable-xxxhdpi-v4/extreme_bass.png
625067        0x989AB         Zip archive data, at least v1.0 to extract, compressed size: 17528, uncompressed size: 17528, name: res/drawable-xxxhdpi-v4/home_theater.png
642672        0x9CE70         Zip archive data, at least v1.0 to extract, compressed size: 100, uncompressed size: 100, name: res/drawable-xxxhdpi-v4/ic_equalizer.png
642848        0x9CF20         Zip archive data, at least v1.0 to extract, compressed size: 24145, uncompressed size: 24145, name: res/drawable-xxxhdpi-v4/ic_launcher.png
667069        0xA2DBD         Zip archive data, at least v1.0 to extract, compressed size: 18923, uncompressed size: 18923, name: res/drawable-xxxhdpi-v4/live_stage.png
686067        0xA77F3         Zip archive data, at least v1.0 to extract, compressed size: 17853, uncompressed size: 17853, name: res/drawable-xxxhdpi-v4/off.png
703989        0xABDF5         Zip archive data, at least v1.0 to extract, compressed size: 15388, uncompressed size: 15388, name: res/drawable-xxxhdpi-v4/stereo.png
719448        0xAFA58         Zip archive data, at least v1.0 to extract, compressed size: 18690, uncompressed size: 18690, name: res/drawable-xxxhdpi-v4/vocalizer.png
738214        0xB43A6         Zip archive data, at least v1.0 to extract, compressed size: 502, uncompressed size: 502, name: res/drawable/connected_device_bluetooth.png
738798        0xB45EE         Zip archive data, at least v1.0 to extract, compressed size: 502, uncompressed size: 502, name: res/drawable/connected_device_mba.png
739374        0xB482E         Zip archive data, at least v1.0 to extract, compressed size: 377, uncompressed size: 377, name: res/drawable/connected_device_mod.png
739825        0xB49F1         Zip archive data, at least v1.0 to extract, compressed size: 833, uncompressed size: 833, name: res/drawable/connected_device_speaker.png
740737        0xB4D81         Zip archive data, at least v1.0 to extract, compressed size: 610, uncompressed size: 610, name: res/drawable/connected_device_wired.png
741422        0xB502E         Zip archive data, at least v1.0 to extract, compressed size: 146843, uncompressed size: 146843, name: res/drawable/disable_audiofx.png
888335        0xD8E0F         Zip archive data, at least v1.0 to extract, compressed size: 131563, uncompressed size: 131563, name: res/drawable/disable_audiofx_land.png
1019971       0xF9043         Zip archive data, at least v1.0 to extract, compressed size: 3952, uncompressed size: 3952, name: res/drawable/effect_bg.png
1023988       0xF9FF4         Zip archive data, at least v1.0 to extract, compressed size: 260, uncompressed size: 260, name: res/drawable/effect_bg_small.png
1024316       0xFA13C         Zip archive data, at least v1.0 to extract, compressed size: 308, uncompressed size: 308, name: res/drawable/ic_audio_effect.png
1024692       0xFA2B4         Zip archive data, at least v1.0 to extract, compressed size: 99, uncompressed size: 99, name: res/drawable/ic_equalizer.png
1024859       0xFA35B         Zip archive data, at least v1.0 to extract, compressed size: 377, uncompressed size: 377, name: res/drawable/ic_equalizer_off.png
1025305       0xFA519         Zip archive data, at least v1.0 to extract, compressed size: 6046, uncompressed size: 6046, name: res/drawable/ic_launcher.png
1031418       0xFBCFA         Zip archive data, at least v1.0 to extract, compressed size: 519244, uncompressed size: 519244, name: resources.arsc
1550712       0x17A978        Zip archive data, at least v2.0 to extract, name: AndroidManifest.xml
1553237       0x17B355        Zip archive data, at least v2.0 to extract, name: classes.dex
```

Now where's the string?
```sh
g -Ua 'Processing ana'
Binary file _bin1-0.extracted/resources.arsc matches
Binary file _bin1-0.extracted/12000.zip matches
Binary file _bin1-0.extracted/0.elf matches
Binary file _bin1-0.extracted/5AC25.zlib matches
```

Okay, so a few files matching, likely from extraction of larger archives containing the sub file, so which is the smallest file?

```sh
ls -lh _bin1-0.extracted/12000.zip _bin1-0.extracted/resources.arsc _bin1-0.extracted/0.elf _bin1-0.extracted/5AC25.zlib
-rw-r--r--  1 samy  staff   2.0M Aug 28 10:06 _bin1-0.extracted/0.elf
-rw-r--r--  1 samy  staff   2.0M Aug 28 10:06 _bin1-0.extracted/12000.zip
-rw-r--r--  1 samy  staff   1.7M Aug 28 10:06 _bin1-0.extracted/5AC25.zlib
-rw-r--r--  1 samy  staff   507K Dec 31  2008 _bin1-0.extracted/resources.arsc
```

Since `resources.arsc` is the smallest file (0.elf looks like an error binwalk, but the zip/zlib make sense to contain an arsc file, especially as APK files for Android apps contain arsc's), and arsc is an Android resource file, we're on the right track. Since it's just a resource file and not an actual binary, it must be used by other binaries, so let's extract the zlib and use the contents of that assuming this is the application we want.

```sh
file ../_bin1-0.extracted/12000.zip
../_bin1-0.extracted/12000.zip: Java archive data (JAR)
```

So need to extract via jar
```sh
jar xf ../_bin1-0.extracted/12000.zip
```

We have some files and a `res` directory, mostly with xml/images
```sh
ls
AndroidManifest.xml
classes.dex
res
resources.arsc
```

Let's take a gander at some of the images:
![apk/res/drawable/connected_device_bluetooth.png](apk/res/drawable/connected_device_bluetooth.png)![apk/res/drawable/connected_device_mba.png](apk/res/drawable/connected_device_mba.png)![apk/res/drawable/connected_device_mod.png](apk/res/drawable/connected_device_mod.png)![apk/res/drawable/connected_device_speaker.png](apk/res/drawable/connected_device_speaker.png)![apk/res/drawable/connected_device_wired.png](apk/res/drawable/connected_device_wired.png)![apk/res/drawable/disable_audiofx.png](apk/res/drawable/disable_audiofx.png)![apk/res/drawable/disable_audiofx_land.png](apk/res/drawable/disable_audiofx_land.png)![apk/res/drawable/effect_bg.png](apk/res/drawable/effect_bg.png)![apk/res/drawable/effect_bg_small.png](apk/res/drawable/effect_bg_small.png)![apk/res/drawable/ic_audio_effect.png](apk/res/drawable/ic_audio_effect.png)![apk/res/drawable/ic_equalizer.png](apk/res/drawable/ic_equalizer.png)![apk/res/drawable/ic_equalizer_off.png](apk/res/drawable/ic_equalizer_off.png)![apk/res/drawable/ic_launcher.png](apk/res/drawable/ic_launcher.png)

Android binaries are generally packed in the dex files so let's unpack via dex2jar
```sh
d2j-dex2jar.sh -f classes.dex
```

Let's unzip the resultant jar.
```sh
mkdir jar && cd jar && unzip ../classes-dex2jar.jar
```

Bunch of java binary files which we can decompile via jad.
```sh
find . | perl -lne '$x||=`pwd`;chomp$x; if(-d $_){chdir($_); system "jad *.class"; chdir($x); }'
```

Let's clean up the code a bit...
```sh
perl -ni -e 'print unless /^\/\/\s*(Decompiled by Jad|Jad home)/' `find . |g jad$`
```

Let's check the directory structure
```sh
 find . |g -v '(class|jad)$'
.
./android
./android/support
./android/support/design
./android/support/design/widget
./android/support/design/internal
./android/support/v7
./android/support/v7/app
./android/support/v7/widget
./android/support/v7/a
./android/support/v7/view
./android/support/v7/view/menu
./android/support/v7/c
./android/support/v7/c/a
./android/support/v7/d
./android/support/v7/d/a
./android/support/v7/e
./android/support/v7/b
./android/support/annotation
./android/support/a
./android/support/v4
./android/support/v4/widget
./android/support/v4/i
./android/support/v4/i/a
./android/support/v4/i/b
./android/support/v4/g
./android/support/v4/a
./android/support/v4/f
./android/support/v4/h
./android/support/v4/c
./android/support/v4/c/a
./android/support/v4/d
./android/support/v4/d/a
./android/support/v4/e
./android/support/v4/b
./android/support/v4/b/a
./android/support/v4/media
./android/support/v4/media/session
./android/support/b
./android/support/b/a
./android/arch
./android/arch/lifecycle
./android/arch/a
./android/arch/a/a
./com
./com/motorola
./com/motorola/audiofx
./com/motorola/audiofx/ui
./com/motorola/audiofx/internal
./com/motorola/audiofx/a
./com/motorola/audiofx/external
./com/motorola/audiofx/external/mod
./com/motorola/audiofx/checkin
./com/motorola/a
```


