---
title: "vmctl 4.0: الملف نفسه على أربعة مُشرفات افتراضية"
date: 2026-09-29
categories: ["cloud"]
tags: ["virtualbox", "libvirt", "kvm", "qemu", "vmware", "iac", "automation", "vm"]
---

عندما [كتبتُ عن vmctl](../vmctl/) قبل أسابيع، كان يقود VirtualBox، وكان libvirt و
QEMU "في خطة الطريق". تلك الجملة كانت تحمل عبئاً كبيراً. الانتقال من مُشرف افتراضي
واحد إلى أربعة لم يكن في جوهره كتابة ثلاث خلفيات إضافية، بل اكتشاف كم كان *ما
أعتقده* عن كل مُشرف افتراضي خاطئاً.

يقود vmctl 4.0 الآن **VirtualBox و libvirt/QEMU-KVM و QEMU المجرّد و VMware
Workstation**. الملف نفسه، والأوامر نفسها، على أيٍّ منها. وهذه التدوينة عمّا كلّفه
ذلك فعلاً، لأن الأجزاء المثيرة للاهتمام لم تكن التي توقّعتها.

## الملف نفسه، أربعة مُشرفات

هذا إعداد واحد:

```yaml
# web-01.yaml
name: web-01
guest_os: ubuntu22.04
cpu:
  count: 4
memory:
  mb: 8192
firmware:
  type: efi64
storage:
  - name: system
    size_mb: 51200
    bus: sata
    bootable: true
networks:
  - network_type: nat
    port_forwards:
      - name: ssh
        host_port: 2222
        guest_port: 22
```

وهذا الملف نفسه أمام كل مُشرف افتراضي، تشغيلاً تجريبياً، كما خرج بالضبط. ولاحظ أن
لا شيء هنا جدول تحويل — كل مُشرف يبني شيئه الأصلي:

```console
$ vmctl -p virtualbox import web-01.yaml --policy nearest
Dry-run mode.  Commands that would be executed:
    1: VBoxManage createvm --name web-01 --ostype Ubuntu22_LTS_64 --register
    2: VBoxManage modifyvm web-01 --memory 8192 --vram 16 --cpus 4 --firmware efi64 ...
    4: VBoxManage createmedium disk --filename ~/VirtualBox VMs/web-01/web-01_system.vdi --size 51200 --format VDI --variant Standard
    7: VBoxManage modifyvm web-01 --natpf1 ssh,tcp,,2222,,22

$ vmctl -p libvirt import web-01.yaml --policy nearest
Dry-run mode.  Commands that would be executed:
    2: qemu-img create -f qcow2 ~/.local/share/libvirt/images/web-01_system.qcow2 51200M
    3: write /tmp/web-01.xml (1080 bytes)
    4: virsh define /tmp/web-01.xml

$ vmctl -p qemu import web-01.yaml --policy nearest
Warning: guest_os: ubuntu22.04 was not applied (QEMU has no guest OS field; it boots
         what the disk contains)
Dry-run mode.  Commands that would be executed:
    2: qemu-img create -f qcow2 ~/.local/share/vmctl/qemu/web-01/web-01_system.qcow2 51200M
    3: write ~/.local/share/vmctl/qemu/web-01/run.sh (629 bytes)

$ vmctl -p vmware import web-01.yaml --policy nearest
Warning: networks[0].port_forwards: tcp *:2222 -> guest:22 was not applied (VMware has
         no per-VM port forwarding; its NAT forwards live in the host-wide vmnetnat.conf)
Dry-run mode.  Commands that would be executed:
    2: vmware-vdiskmanager -q -c -s 51200MB -a lsilogic -t 0 ~/Documents/Virtual Machines/web-01/web-01_system.vmdk
    3: write ~/Documents/Virtual Machines/web-01/web-01.vmx (930 bytes)
```

يحصل VirtualBox على قرص VDI وقاعدة تمرير منفذ في NAT. ويحصل libvirt على qcow2
ووثيقة XML للنطاق. ويحصل QEMU المجرّد على مجلّد فيه سطر أوامر قابل للتنفيذ — فلهذا
المُشرف، السكربت *هو* الجهاز؛ لا خدمة ولا سجل. ويحصل VMware على VMDK وملف `.vmx`.

**والتحذيرات هي المقصد.** المُشرفات الأربعة لا تملك الميزات نفسها، والأمانة تقتضي
قول ذلك لكل حقل بدلاً من التظاهر. QEMU لا يملك مكاناً يسجّل فيه نظام الضيف. وتمريرات
NAT في VMware على مستوى المضيف لا الجهاز، فلا يمكن تنفيذ القاعدة — ويقول أي قاعدة
ولماذا. و ``--policy strict`` يرفض بدلاً من الاستبدال، و ``--policy nearest`` يستبدل
ويخبرك بما غيّره.

## جداول القدرات مقيسة، لا مقروءة من التوثيق

هذا الجزء لم أكن لأتوقّعه. كل جواب عن "هل يستطيع هذا المُشرف ذلك؟" في vmctl أُرسي
عبر *سؤال المنتج العامل*، وكل مُشرف يُعلن من أين جاءت أرقامه:

```console
$ vmctl -p libvirt capabilities
...
buses:
  bus           disk    cdrom   floppy  ports
  ide           yes     yes     -       1-2 x 2
  sata          yes     yes     -       1-6   <- native for cdrom
  usb           yes     yes     -       1-8
  virtio-blk    yes     yes     -       1-32   <- native for disk
  virtio-scsi   yes     yes     -       1-256

evidence: probed on libvirt 11.10.0 / QEMU 10.1.0, machine q35. ... For libvirt,
define-time acceptance is evidence of nothing: it takes a vmxnet3 or a VMDK happily
and then fails to start the domain.
```

اقرأ الجملة الأخيرة مرّةً ثانية، فقد كلّفتني يوماً كاملاً. **libvirt يقبل تعريف
النطاق ثم يفشل في تشغيله.** فإن بنيت جدول قدرات بفحص ما يقبله ``virsh define``،
حصلت على جدول خاطئ بطريقة لا تظهر إلّا لاحقاً، على آلة شخص آخر. والاختبار الموثوق
الوحيد كان تعريف نطاق *وتشغيله*، واحداً لكل تركيبة، وتسجيل ما نجا.

ونتيجة ملموسة: **لا توجد صيغة صور أقراص يستطيع الأربعة جميعاً إنشاءها.** ينشئ
VirtualBox سبعاً، و libvirt و QEMU اثنتين لكلٍّ، و VMware واحدة بالضبط، والتقاطع
فارغ. فالإعداد الذي يحدّد ``format: qcow2`` قابل للنقل إلى اثنين من الأربعة؛ وإن
تركت الصيغة استخدم كل مُشرف صيغته. و vmctl يعرف أيّها أيّ، لكل مضيف، لأنه قاس.

## الانحراف: ما تغيّر بعد أن كتبت الملف

التدوينة الأولى كانت: صِف ← عايِن ← طبّق. وما كان ناقصاً هو ما يحدث بعد ثلاثة
أسابيع، حين ينقر أحدهم شيئاً في واجهة رسومية:

```console
$ vmctl diff web-01 web-01.yaml
web-01 vs web-01.yaml: 1 changed
  ~ memory.mb  vm 8192  file 16384

$ vmctl apply web-01.yaml --execute
web-01 differs from the file in 1 place(s)
  ~ memory.mb  vm 8192  file 16384
Applied web-01.yaml to 'web-01'
    1: write /tmp/web-01.xml (1269 bytes)
    2: virsh define /tmp/web-01.xml
```

يُغيّر ``apply`` **ما انحرف فقط**، لا الجهاز كله، ولن يُعيد إنشاء قرص يملكه الجهاز
أصلاً. وهذا الجزء الأخير كان عِلّة يوماً، وعِلّة مكلّفة: نسخة مبكّرة من ``edit``
كانت تُعيد إنشاء الأقراص، وهو ما كان يعني على بعض المُشرفات حذف البيانات. وهناك الآن
قاعدة واحدة في ذلك، واختبار مطابقة على كل مُشرف أن يجتازه.

ويقارن ``diff`` كذلك ما يذكره الملف *فعلاً* فقط. فالقيمة الافتراضية ليست طلباً،
والملف الذي لا يذكر ``vram_mb`` لا يُبلّغ عن انحراف لأن المُشرف اختار رقماً.

## التغيير دون تعديل الملف

كان ``import`` يقبل اسماً جديداً وصيغة قرص، ولا شيء غير ذلك — فالشيئان اللذان يريد
أي أحد تغييرهما فعلاً لم يكن ممكناً تغييرهما:

```console
$ vmctl import web-01.yaml --set memory.mb=4096 --set cpu.count=8
$ vmctl import web-01.yaml --add-disk size_mb=40960,bus=virtio-blk,format=qcow2
$ vmctl import web-01.yaml --add-nic network_type=bridged,model=virtio
$ vmctl import web-01.yaml --patch bigger.yaml
```

وهذه دمجٌ فوق الإعداد *قبل* أن يصبح جهازاً، بالدمج نفسه الذي يستخدمه ``extends:``
لوراثة الملفات — فالحقل المُبدَّل يُتحقَّق منه ويُفحص أمام القدرات ويُترجم كأي حقل
مكتوب تماماً. و ``--add-disk format=vdi`` أمام VMware يُرفَض في ``strict`` ويُستبدَل
في ``nearest`` بلا أي معالجة خاصة في أي مكان.

وعندما يتعارض علَمان، يقول أيّهما فاز بدلاً من الاختيار بصمت:

```console
note: storage[1].format is 'qcow2' from --add-disk, which outranks --disk-format ('vdi')
```

## نقل جهاز إلى مُشرف افتراضي آخر

```console
$ vmctl migrate web-01 --to qemu --policy nearest
web-01 (libvirt) -> web-01 (qemu)
Configuration only: the new VM gets blank disks. Pass --with-disks to bring the data.
Warning: firmware.secure_boot: True was not applied (secure boot needs an OVMF
         variables file per VM)
```

الإعداد افتراضياً، والأقراص عند الطلب، وبيان صريح بكل ما لم يَنجُ من الرحلة.

## هل تعمل هذه الآلة أصلاً؟

أمران أستخدمهما أكثر مما توقّعت. يُجيب ``doctor`` عن "هل يستطيع هذا المضيف تشغيل
جهاز افتراضي، ولماذا هو بطيء":

```console
$ vmctl doctor
provider: libvirt
[ok  ] python: 3.13
[--  ] hardware virtualisation: unavailable; guests will be emulated and slow
         hint: on a nested setup, enable VT-x/AMD-V for this VM; otherwise check that
               /dev/kvm exists and you are in the group that may use it
[ok  ] virsh: /usr/bin/virsh
[ok  ] libvirt version: 11.10.0
[ok  ] connection works: yes, 0 domain(s)
[--  ] domain type: qemu (emulated)
```

ويُثبت ``selftest`` أن المُشرف الافتراضي يوافق vmctl، بإنشاء جهاز زائل واستخدامه ثم
حذفه — عشر فحوص، على عتاد حقيقي، في ثوانٍ.

واللقطات موجودة أيضاً، مع حقل الوصف الذي تُسقطه معظم الأدوات:

```console
$ vmctl snapshot take web-01 before-upgrade -d "kernel 6.9, rolling back if it panics"
Took snapshot 'before-upgrade' of 'web-01'

$ vmctl snapshot list web-01
* before-upgrade  2026-09-29 19:26:44 +0000  shutoff  -- kernel 6.9, rolling back if it panics
```

## الجزء غير المريح: الاختبارات ليست دليلاً

أريد أن أكون صريحاً في كيف وُجدت العلل الحقيقية في هذه الأداة، لأن ذلك غيّر طريقة
عملي.

في لحظة ما كان لـ vmctl **1194 اختباراً ناجحاً** وأمر ``selftest`` يُبلّغ 10/10 أمام
مُشرفين افتراضيين حقيقيين. ومع كل ذلك الأخضر، كان هذا صحيحاً:

```console
$ vmctl export web-01 -o web-01.json     # ملف .json
$ head -1 web-01.json
name: web-01                              # ...يحتوي YAML
```

كان لـ ``--format`` قيمة افتراضية، فلم يستطع الأمر التمييز بين "لم يُذكر" و"ذُكر
كـ yaml"، ولم يعمل الاستنباط الذي وعد به نصّ مساعدته أبداً. ولم يمسكه أي اختبار لأن
لا اختبار طلب ملف ``.json`` من قبل دون أن يُمرّر ``--format`` أيضاً. كانت مجموعة
اختباراتي و ``selftest`` مكتوبتين كلتاهما أمام تصوّري أنا للأداة، فحين كان التصوّر
خاطئاً اتّفقتا ونجحتا.

فكتبتُ أداة تقود سطر الأوامر كما يقوده الإنسان، على مُشرفات حقيقية، **وتقرأ الملفات
التي تنزل على القرص**: لكل صيغة صورة يستطيع كل مُشرف إنشاءها، على كل ناقل يحمل
قرصاً، أنشئ جهازاً حقيقياً، وصدّره، وأعد استيراده، وقارن التصدير بالجهاز، واحذفه،
وتأكّد أنه زال. أحد وعشرون جهازاً حقيقياً، و336 فحصاً، والمُشرفات الأربعة جميعاً —
VirtualBox و VMware عبر SSH إلى مضيف Windows، و libvirt و QEMU محلياً.

وجدت خمس علل في عصر واحد. ثلاث منها كسرت الرحلة الدائرية، وهي الشيء الوحيد الذي
توجد الأداة من أجله:

- تصدير جهاز libvirt وإعادة إنشائه **والأصل ما زال موجوداً** كان مستحيلاً: التصدير
  يحمل مُعرِّف UUID للنطاق، فيرفض libvirt النطاق الثاني
- جهاز QEMU بقرصين على ``virtio-scsi`` أو ``usb`` لم يكن يُعاد استيراده، لأن المُصدِر
  لم يسجّل أيّ LUN لكل قرص فقُرئ الاثنان على المنفذ 0
- على Windows، كتب ``export`` الملف صحيحاً ثم **مات وهو يطبع أنه كتبه** —
  ``UnicodeEncodeError`` على حرف سهم لا تملك صفحة ترميز الطرفية مكاناً له

ولم يكن أيٌّ منها مرئياً لمجموعة الاختبارات. وظهرت علّتان أُخريان لاحقاً بالطريقة
نفسها، إحداهما انحدار أطلقتُه في إصدارين: طبع ``vmctl providers`` جدولاً فارغاً، لأن
السجل كان يعتمد على استيراد غير ذي صلة كنتُ قد أزلته. والاختبار الذي يؤكّد أن
المُشرفات مُسجَّلة كان هو نفسه يستورد الوحدة التي تُسجّلها.

والتقرير مُولَّد ومحفوظ في المستودع، فلا يمكن أن ينحرف عمّا سجّلته التشغيلات فعلاً،
ومعه تسجيل asciinema لكل مُشرف.

## إن أردت أن ترى ما تفعله عنك

كتبتُ صفحة توثيق تبني جهازاً واحداً بـ ``virsh`` و ``qemu-img`` وحدهما — القرص،
ووثيقة XML من 38 سطراً، والتعريف، والتشغيل، وتغيير الذاكرة، والتفكيك — ثم تبني
الجهاز نفسه بـ vmctl:
[**إنشاء جهاز افتراضي بـ virsh يدوياً**](https://abdelhaleemahmed.github.io/vmctl/docs/ar/guide/virsh-by-hand.html).
وكل أمر وخطأ في تلك الصفحة نُفِّذ فعلاً.

وثلاثة أمور منها تستحق المعرفة سواء استخدمتَ vmctl أم لا:

- يحفظ libvirt وثيقةً *مختلفة* عن التي تسلّمها — 38 سطراً داخلاً و146 خارجاً — فلا
  يُطابق ``dumpxml`` ملفك أبداً ولا يمكن مقارنتهما
- يُقبل ``setmaxmem --config`` على نطاق عامل ويُغيّر التعريف المحفوظ وحده، فتختلف
  القيمة الدائمة عن العاملة بلا أي تحذير
- يترك ``undefine --remove-all-storage`` **القرص موجوداً** حين لا تكون الصورة في
  مجمع تخزين لـ libvirt، ويُبلّغ بذلك، ويُزيل تعريف النطاق على أي حال، ويخرج بالرمز 0

يُصدر vmctl أوامر ``virsh``؛ وهو ليس بديلاً عن معرفتها. فمجمعات التخزين وشبكات
libvirt والترحيل الحيّ والتوصيل أثناء التشغيل كلها ما زالت من عمل ``virsh``.

## للحصول عليه

vmctl برخصة MIT ويحتاج **Python 3.13 أو أحدث** — وهذا الحد تحرّك، وتحرّك عن قصد،
لأن دعم 3.8 كان هو نفسه يُنتج عللاً. وهو ليس على PyPI، فثبّت الحزمة من الإصدار:

```bash
pip install https://github.com/abdelhaleemahmed/vmctl/releases/download/v4.0.3/vmctl-4.0.3-py3-none-any.whl
```

ثم، قبل أي شيء آخر:

```bash
vmctl doctor      # ما تستطيعه هذه الآلة
vmctl selftest    # إثبات أن المُشرف يوافق، على جهاز زائل
```

- **الشيفرة:** https://github.com/abdelhaleemahmed/vmctl
- **التوثيق:** https://abdelhaleemahmed.github.io/vmctl/ — بالإنجليزية والعربية
- **الإصدارات:** https://github.com/abdelhaleemahmed/vmctl/releases

وملاحظة إن كنت قرأت التدوينة الأولى: صار ``--apply`` الآن ``--execute``. كان لا بدّ
أن يتغيّر، لأن ``apply`` أصبح أمراً مستقلاً بذاته، وكلمة واحدة لا تصلح لتعني "نفّذ
هذه الخطة" و"وائم هذا الجهاز" في الوقت نفسه.
