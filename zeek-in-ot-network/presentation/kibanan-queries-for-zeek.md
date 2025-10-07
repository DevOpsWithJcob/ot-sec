### کیس‌های مشابه برای تشخیص دستورات غیرمجاز ICS (OT-specific)

کوئری اصلی شما بر تشخیص دستورات نوشتن (WRITE) غیرمجاز در پروتکل Modbus تمرکز دارد، که در محیط‌های OT/ICS (مانند کارخانه‌ها یا نیروگاه‌ها) برای شناسایی حملاتی مثل دستکاری PLCها استفاده می‌شود. در ادامه، ۳ کیس مشابه برای پروتکل‌های دیگر ICS (DNP3، S7comm و Ethernet/IP) ارائه می‌دهم. هر کیس شامل کوئری KQL (Kibana Query Language) برای Kibana، توضیح مختصر، و نمونه لاگ مخرب (بر اساس لاگ‌های Zeek واقعی یا شبیه‌سازی‌شده از منابع معتبر مانند CISA ICSNPP و Elastic Blog) است. این کوئری‌ها بر اساس ایندکس‌های Zeek (مانند `zeek-*`) طراحی شده‌اند و می‌توانند در Kibana برای threat hunting استفاده شوند.

این کیس‌ها بر اساس پروتکل‌های رایج ICS (از منابع مانند ، ،  و ) انتخاب شده‌اند، که Zeek از طریق پلاگین‌های ICSNPP پشتیبانی می‌کند.

#### ۱. **کیس DNP3: تشخیص دستورات نوشتن غیرمجاز (Write Objects) در پروتکل DNP3**
   - **کوئری KQL**:
     ```
     event.module: zeek AND event.dataset: zeek.dnp3 AND dnp3.func_code: WRITE_OBJECT OR dnp3.func_code: NON_OPERATIONAL_REQUEST AND dnp3.iin: 0x0000 AND dnp3.objects.group: 12 OR dnp3.objects.variation: 1
     ```
     - **توضیح**: این کوئری دستورات نوشتن به اشیاء (مانند کنترل relays در شبکه‌های برق) را در DNP3 لاگ (dnp3.log) شناسایی می‌کند. شرط `iin: 0x0000` نشان‌دهنده عدم خطا (successful write) است، که می‌تواند نشانه حمله reconnaissance یا دستکاری باشد. در OT، DNP3 برای SCADA در بخش انرژی استفاده می‌شود و این کوئری spikes ناگهانی writes را فلگ می‌کند. (بر اساس  و ، از پلاگین ICSNPP-DNP3 برای لاگ‌های پیشرفته استفاده کنید.)
   - **نمونه لاگ مخرب (از dnp3.log)**:
     ```
     #fields ts uid id.orig_h id.orig_p id.resp_h id.resp_p proto iin func_code objects
     1728211200.123456 D987654321 10.0.0.1 20000 10.0.0.2 20000 tcp 0x0000 WRITE_OBJECT [group=12, variation=1, points=1-5, value=0xFF]
     ```
     - **چرا مخرب است؟**: این لاگ نشان‌دهنده نوشتن موفق به گروه 12 (binary outputs) بدون exception است، که می‌تواند هکر را قادر به تغییر وضعیت relays (مثل قطع برق) کند. در سناریوی حمله، IP منبع (10.0.0.1) می‌تواند از یک دستگاه آلوده باشد، و UID برای correlation با conn.log استفاده شود. (نمونه از  و شبیه‌سازی بر اساس ICSNPP logs.)

#### ۲. **کیس S7comm: تشخیص بلوک‌های نوشتن غیرمجاز (Write Blocks) در پروتکل S7comm**
   - **کوئری KQL**:
     ```
     event.module: zeek AND event.dataset: zeek.s7comm AND s7comm.function: WRITE_VAR_REQUEST OR s7comm.subfunction: WRITE_MULTIPLE_BLOCKS AND s7comm.error_code: 0 AND s7comm.pdu_type: 0x01
     ```
     - **توضیح**: این کوئری درخواست‌های نوشتن متغیرها (write var request) یا بلوک‌های متعدد را در لاگ s7comm.log شناسایی می‌کند. شرط `error_code: 0` نشان‌دهنده موفقیت است، که در ICS برای PLCهای Siemens (S7) خطرناک است. در OT، این برای تشخیص دستکاری برنامه‌های PLC (مثل تغییر لاجیک کنترل) مفید است. (از پلاگین ICSNPP-S7comm در  و  الهام‌گرفته؛ برای لاگ‌های COTP هم قابل گسترش است.)
   - **نمونه لاگ مخرب (از s7comm.log)**:
     ```
     #fields ts uid id.orig_h id.orig_p id.resp_h id.resp_p proto function subfunction error_code pdu_type
     1728211300.456789 S123456789 192.168.2.50 102 192.168.2.100 102 tcp WRITE_VAR_REQUEST WRITE_MULTIPLE_BLOCKS 0 0x01
     ```
     - **چرا مخرب است؟**: این لاگ نوشتن موفق بلوک‌های متعدد بدون خطا را نشان می‌دهد، که می‌تواند هکر را قادر به آپلود کد مخرب به PLC کند (مثل تغییر سرعت موتور). IP منبع (192.168.2.50) ممکن است از HMI آلوده باشد، و pdu_type 0x01 تأیید درخواست معتبر اما غیرمجاز است. (نمونه از  و شبیه‌سازی بر اساس GitHub ICSNPP-S7comm.)

#### ۳. **کیس Ethernet/IP (CIP): تشخیص دستورات نوشتن تگ‌های غیرمجاز (Tag Writes) در پروتکل CIP**
   - **کوئری KQL**:
     ```
     event.module: zeek AND event.dataset: zeek.enip AND enip.service_code: 4 (Unconnected Message) OR enip.service_code: 20 (Multiple Service Packet) AND cip.tag_write: true AND cip.error_code: 0
     ```
     - **توضیح**: این کوئری سرویس‌های نوشتن تگ (tag write) در ENIP/CIP لاگ (enip.log) را فلگ می‌کند، با تمرکز روی unconnected messages بدون خطا. در OT، CIP برای اتوماسیون صنعتی (مثل Rockwell PLCها) استفاده می‌شود و این کوئری حملاتی مثل تغییر مقادیر سنسورها را تشخیص می‌دهد. (بر اساس  و ، از پلاگین Amazon Zeek برای ENIP.)
   - **نمونه لاگ مخرب (از enip.log)**:
     ```
     #fields ts uid id.orig_h id.orig_p id.resp_h id.resp_p proto service_code tag_write error_code
     1728211400.789012 E456789012 172.16.1.10 44818 172.16.1.20 44818 tcp 4 true 0
     ```
     - **چرا مخرب است؟**: این لاگ نوشتن موفق تگ بدون خطا را در unconnected message نشان می‌دهد، که هکر می‌تواند برای دستکاری مقادیر تگ (مثل دما یا فشار در یک فرآیند) استفاده کند. IP منبع (172.16.1.10) می‌تواند از یک attacker خارجی باشد، و service_code 4 نشان‌دهنده عدم اتصال معتبر اما مخرب است. (نمونه از  و شبیه‌سازی بر اساس Corelight ICS/OT logs.)

### نکات کلی برای استفاده
- **تنظیم در Kibana**: این کوئری‌ها را در Discover وارد کنید و ایندکس `zeek-*` را انتخاب کنید. برای آلارم، از Threshold Rule در Elastic Security استفاده کنید (مثل >10 write در 1 دقیقه).
- **پلاگین‌های Zeek**: برای DNP3 و S7comm، پلاگین‌های ICSNPP (از  و ) را نصب کنید تا لاگ‌های دقیق‌تری تولید شود.
- **تست**: از PCAPهای CIC ICS Dataset یا GitHub ICSNPP برای شبیه‌سازی استفاده کنید.
- **منابع**: این کیس‌ها بر اساس ، ، ،  و  تدوین شده‌اند.

اگر نیاز به کوئری‌های بیشتر (مثل BACnet) یا تنظیم Logstash دارید، بگویید!