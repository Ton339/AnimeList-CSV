# Ckan-weeb: วิธี Setup ขั้นที่ 2 และขั้นที่ 4

ระบบบัญชีข้อมูล: **Ckan-weeb** (ศูนย์รวมข้อมูลอนิเมะ | Anime Data Hub)
สภาพแวดล้อม: CKAN 2.11.6 ติดตั้งผ่าน Docker (ckan-docker), เข้าใช้งานที่ `https://localhost:8443`

---

## ขั้นที่ 2: การนำข้อมูลเข้าสู่ระบบ

ผลที่ได้: **มีการนำเข้าข้อมูล** (Dataset: `AnimeList` ในองค์กร `MyAnimeList`)

### 2.1 ล็อกอินด้วยบัญชี sysadmin

ดูชื่อผู้ใช้และรหัสผ่านจากไฟล์ `.env`

```
CKAN_SYSADMIN_NAME=...
CKAN_SYSADMIN_PASSWORD=...
```

เข้า `https://localhost:8443` แล้วกด **Log in**

### 2.2 สร้าง Organization

1. เมนู **Organizations** → **Add Organization**
2. Name: `MyAnimeList`
3. Description: ข้อมูลอนิเมะจากชุดข้อมูล MyAnimeList
4. กด **Create Organization**

### 2.3 สร้าง Dataset

1. เมนู **Datasets** → **Add Dataset**
2. Title: `AnimeList` (URL จะเป็น `animelist`)
3. Description: รายชื่ออนิเมะและรายละเอียดจากชุดข้อมูล MyAnimeList
4. Organization: `MyAnimeList`
5. Visibility: **Public**
6. กด **Next: Add Data**

### 2.4 เพิ่ม Resource (ข้อมูล CSV)

ในระบบนี้ปุ่ม Upload ไม่แสดงในหน้า Add resource จึงนำเข้าด้วย **Link** แทน

**ข้อควรระวัง:** ลิงก์ Kaggle ใช้ไม่ได้ เพราะเป็นหน้าเว็บและต้องล็อกอินก่อนดาวน์โหลด ต้องใช้ลิงก์ตรงของไฟล์ (raw) จาก GitHub

1. ดาวน์โหลด `AnimeList.csv` จาก Kaggle แล้วอัปโหลดขึ้น GitHub (repo ต้องเป็น Public)
2. ลิงก์ในหน้า GitHub (`.../blob/main/AnimeList.csv`) ต้องแปลงเป็นลิงก์ raw:

   ```
   https://raw.githubusercontent.com/Ton339/AnimeList-CSV/main/AnimeList.csv
   ```

3. ทดสอบวางลิงก์ในเบราว์เซอร์ ต้องเห็นข้อความ CSV หรือเริ่มดาวน์โหลดทันที
4. ในหน้า Add resource กดปุ่ม **Link** แล้ววาง URL ข้างบน
5. Name: `AnimeList CSV`
6. Format: `CSV`
7. กด **Finish**

### 2.5 แก้ error "Resource too large to download"

อาการ: แท็บ DataStore ขึ้น `Resource too large to download: 3130127 > max (1024000)`
สาเหตุ: ไฟล์ 3.1 MB เกินลิมิตเริ่มต้นของ datapusher (1,024,000 bytes) การนำเข้า dataset ยังสำเร็จ แต่ตารางข้อมูลใน DataStore ยังไม่ถูกสร้าง

**ขั้นตอนแก้**

1. เพิ่มค่าในไฟล์ `.env`

   ```
   CKAN__DATAPUSHER__MAX_CONTENT_LENGTH=100000000
   DATAPUSHER_MAX_CONTENT_LENGTH=100000000
   ```

2. ถ้า service `datapusher` ใน `docker-compose.yml` กำหนด `environment:` เป็นรายตัว ให้เพิ่มบรรทัดนี้ด้วย

   ```yaml
       environment:
         - DATAPUSHER_MAX_CONTENT_LENGTH=${DATAPUSHER_MAX_CONTENT_LENGTH}
   ```

3. สร้าง container ใหม่ (ต้อง recreate ไม่ใช่แค่ restart เพื่อให้อ่าน `.env` ใหม่)

   ```
   docker compose up -d --force-recreate ckan datapusher
   ```

4. ตรวจว่าค่าเข้า container แล้ว

   ```
   docker compose exec ckan env | findstr MAX_CONTENT
   docker compose exec datapusher env | findstr MAX
   ```

   ต้องเห็น `100000000` ทั้งสองฝั่ง

5. เปิดหน้า Resource → แท็บ **DataStore** → กด **Upload to DataStore** แล้วรอ 10-30 วินาที

### 2.6 ตรวจสอบผล

- [ ] หน้า `https://localhost:8443/dataset/animelist` แสดง resource `AnimeList CSV`
- [ ] Dataset มีสถานะ Active และเป็น Public
- [ ] (ถ้าโหลด DataStore สำเร็จ) เห็นตารางข้อมูลในหน้า Resource
- [ ] เรียก API ได้ผล `"success": true`

  ```
  https://localhost:8443/api/3/action/datastore_search?resource_id=<RESOURCE_ID>&limit=5
  ```

**หลักฐานที่ควรแคป:** หน้า Dataset, หน้า Resource, ผล API

---

## ขั้นที่ 4: การปรับแต่ง CKAN ให้มีความสวยงาม

ธีม: ม่วง-ชมพู (ซากุระยามค่ำ) สอดคล้องกับเนื้อหาอนิเมะ
ทำครบ **11 รายการ** แบ่งเป็น 2 กลุ่ม

| กลุ่ม | ข้อ | วิธีทำ |
|---|---|---|
| A. หน้า Admin | 1, 2, 3, 4, 5, 6, 10 | แก้ที่ `https://localhost:8443/ckan-admin/config` |
| B. ไฟล์ template/รูป | 7, 8, 9, 11 | mount โฟลเดอร์ `custom/` เข้า container |

### กลุ่ม A: หน้า Admin

เข้า `https://localhost:8443/ckan-admin/config` (ต้องเป็น sysadmin)

#### ข้อ 1: เปลี่ยนชื่อไซต์

| ช่อง | ค่า |
|---|---|
| Site Title | `Ckan-weeb` |
| Site Tag Line | `ศูนย์รวมข้อมูลอนิเมะ \| Anime Data Hub` |

#### ข้อ 2: เปลี่ยนรูปโลโก้

ใช้ไฟล์ `logo.svg` (ตัวอักษรขาวบนพื้นโปร่งใส ใช้กับหัวเว็บสีเข้ม)

- ถ้ามีปุ่ม Upload: อัปโหลด `logo.svg` ที่ช่อง **Site Tag Logo**
- ถ้าไม่มีปุ่ม: ใส่ URL ลงในช่องเดียวกัน เช่น `/images/logo.svg` (ไฟล์อยู่ในโฟลเดอร์ `custom/public/images/` ตามกลุ่ม B) หรือ URL จาก GitHub

#### ข้อ 3: เปลี่ยนธีมสีของเว็บไซต์

ช่อง **Style** เลือก `Default` แล้วกำหนดสีด้วย Custom CSS (ข้อ 4)

#### ข้อ 4: ปรับแต่งหน้าหลักโดยใช้ CSS

วางในช่อง **Custom CSS**

```css
/* ===== ธีม Ckan-weeb: ม่วง-ชมพู ===== */

/* แถบบนและหัวเว็บ */
.account-masthead { background-color: #1a0b3d; }
.masthead { background-color: #2b1055; background-image: none; }

/* กล่องค้นหาหน้าแรก */
.module-search .module-content,
.homepage .module-search .module-content { background-color: #7b2ff7; }

/* หัวข้อกล่อง (เช่น Popular tags) */
.module-heading { background-color: #2b1055; color: #fff; }

/* ปุ่มและลิงก์ */
.btn-primary { background-color: #ff4d8d; border-color: #ff4d8d; color: #fff; }
.btn-primary:hover { background-color: #e0306f; border-color: #e0306f; }
a { color: #7b2ff7; }
a:hover { color: #ff4d8d; }

/* เมนูหลัก */
.masthead .navigation .nav-pills li a:hover,
.masthead .navigation .nav-pills li.active a { background-color: #ff4d8d; }

/* ===== ข้อ 10: ภาพพื้นหลัง (ลายจุดซากุระ + ไล่สี) ===== */
body {
  background-color: #fdf2f8;
  background-image:
    radial-gradient(circle at 20px 20px, rgba(255, 77, 141, 0.12) 3px, transparent 4px),
    linear-gradient(135deg, #fdf2f8 0%, #ede7ff 100%);
  background-size: 48px 48px, 100% 100%;
  background-attachment: fixed;
}
```

หมายเหตุ: ถ้าส่วนไหนสีไม่เปลี่ยน ให้กด F12 → Inspect ดูชื่อ class จริง แล้วปรับ selector

#### ข้อ 5: เปลี่ยนข้อความเกริ่นนำ

วางในช่อง **Intro Text** (รองรับ Markdown)

```markdown
## ยินดีต้อนรับสู่ Ckan-weeb 🌸

**Ckan-weeb** คือศูนย์รวมข้อมูลอนิเมะแบบเปิด สร้างบนแพลตฟอร์ม CKAN
รวบรวมข้อมูลรายชื่ออนิเมะ คะแนนรีวิว แนวเรื่อง สตูดิโอผู้ผลิต
และรายการอนิเมะของผู้ใช้ เพื่อให้ค้นหา ดูตัวอย่าง และดาวน์โหลดไปวิเคราะห์ต่อได้ง่าย

ลองค้นหาจากช่อง **Search data** ด้านขวา เช่น `Naruto`, `Action` หรือ `Studio`
```

#### ข้อ 6: เปลี่ยนข้อความเกี่ยวกับระบบ

วางในช่อง **About**

```markdown
## เกี่ยวกับ Ckan-weeb

Ckan-weeb เป็นระบบบัญชีข้อมูลสำหรับรวบรวมและเผยแพร่ข้อมูลอนิเมะ
พัฒนาขึ้นเพื่อการเรียนรู้การติดตั้งและใช้งานระบบ CKAN

### ข้อมูลที่มีในระบบ
- **AnimeList** ข้อมูลรายชื่ออนิเมะจากชุดข้อมูล MyAnimeList Dataset
- จัดกลุ่มตามหน่วยงาน (Organization) และหมวดหมู่ (Group)
- เรียกใช้ผ่าน CKAN Data API ได้ (Create, Read, Update, Delete)

### เทคโนโลยี
CKAN 2.11 · Docker · PostgreSQL · Solr · DataStore

### ติดต่อ
อีเมล: your-email@example.com
```

#### ข้อ 10: เปลี่ยนภาพพื้นหลัง

ทำในข้อ 4 แล้ว (บล็อก `body { ... }` ใน Custom CSS)

กด **Update Config** แล้ว Ctrl+F5 เพื่อดูผล

### กลุ่ม B: ข้อ 7, 8, 9, 11

ไฟล์ทั้งหมดอยู่ในโฟลเดอร์ `custom/`

```
custom/
├── templates/
│   ├── footer.html                      # ข้อ 7, 8
│   └── home/snippets/promoted.html      # ข้อ 9
└── public/images/
    ├── banner.svg / banner.png          # ข้อ 9 (420 x 220)
    ├── favicon.ico / favicon.png        # ข้อ 11
    └── logo.svg                         # ข้อ 2
```

#### ขั้นที่ 1: วางโฟลเดอร์

แตกไฟล์ให้โฟลเดอร์ `custom/` อยู่ข้างไฟล์ `docker-compose.yml` ของโปรเจกต์ (`E:\work\school\Team\ckan-weeb\custom`)

#### ขั้นที่ 2: mount เข้า container

ใน `docker-compose.yml` ที่ service `ckan` ส่วน `volumes:` เพิ่ม

```yaml
      - ./custom/templates:/srv/app/custom_templates
      - ./custom/public:/srv/app/custom_public
```

#### ขั้นที่ 3: ตั้งค่าใน `.env`

```
CKAN___EXTRA_TEMPLATE_PATHS=/srv/app/custom_templates
CKAN___EXTRA_PUBLIC_PATHS=/srv/app/custom_public
CKAN__FAVICON=/images/favicon.ico
```

(ใช้ขีดล่างสามตัวหลัง `CKAN` เพราะเป็นตัวแปรที่ไม่มี prefix `ckan.` และต้องเปิด plugin `envvars` ใน `CKAN__PLUGINS`)

#### ขั้นที่ 4: สร้าง container ใหม่

```
docker compose up -d --force-recreate ckan
```

รอ 30-60 วินาที แล้วกด Ctrl+F5

#### ข้อ 7: เปลี่ยนที่อยู่ เบอร์โทร อีเมลใน Footer

แก้ไฟล์ `custom/templates/footer.html` ในส่วน `weeb-footer-contact`

```html
<address>
  ชื่อหน่วยงานของคุณ<br>
  เลขที่ 123 ถนนตัวอย่าง ตำบลตัวอย่าง อำเภอตัวอย่าง จังหวัดตัวอย่าง 20000<br>
  โทรศัพท์: 038-000-000<br>
  อีเมล: <a href="mailto:your-email@example.com">your-email@example.com</a>
</address>
```

เปลี่ยนข้อความตัวอย่างเป็นข้อมูลจริง แล้ว `docker compose restart ckan`

#### ข้อ 8: ใส่ลิงก์ไปยังเว็บไซต์ต่าง ๆ ใน Footer

ในไฟล์เดียวกัน ส่วน "ลิงก์ที่เกี่ยวข้อง" เพิ่มหรือลบรายการ `<li>` ได้

```html
<li><a href="https://ckan.org" target="_blank" rel="noopener">CKAN.org</a></li>
<li><a href="https://docs.ckan.org" target="_blank" rel="noopener">CKAN Documentation</a></li>
<li><a href="https://myanimelist.net" target="_blank" rel="noopener">MyAnimeList</a></li>
<li><a href="https://data.go.th" target="_blank" rel="noopener">data.go.th</a></li>
```

#### ข้อ 9: เปลี่ยนรูปแบนเนอร์ (420 x 220)

รูปสีเทา "420 x 220" และข้อความ "This is a featured section" อยู่ในไฟล์ `home/snippets/promoted.html` ซึ่งถูกแทนที่ด้วยไฟล์ใหม่ใน `custom/templates/home/snippets/promoted.html`

บรรทัดรูปแบนเนอร์

```html
<img src="{{ h.url_for_static('/images/banner.svg') }}" alt="Ckan-weeb Anime Data Hub">
```

ขนาดควบคุมด้วย CSS ที่อยู่ในไฟล์เดียวกัน

```css
.weeb-banner img {
  width: 420px;
  height: 220px;
  max-width: 100%;
  object-fit: cover;
}
```

ตรวจว่ามี mount ถูกต้อง: เปิด `https://localhost:8443/images/banner.svg` ต้องเห็นรูป

#### ข้อ 11: เปลี่ยนรูปไอคอนของเว็บไซต์ (favicon)

วางไฟล์ `favicon.ico` ไว้ที่ `custom/public/images/` และตั้ง `CKAN__FAVICON=/images/favicon.ico` ใน `.env` (ตามขั้นที่ 3) เบราว์เซอร์แคช favicon หนัก ถ้าไม่เปลี่ยนให้ปิดแล้วเปิดแท็บใหม่

### การแก้ปัญหา

| อาการ | วิธีแก้ |
|---|---|
| `/images/banner.svg` ไม่ขึ้นรูป | เช็ค `docker compose exec ckan env \| findstr EXTRA` ต้องเห็นทั้ง 2 ตัวแปร |
| แบนเนอร์/footer ไม่เปลี่ยน | `docker compose restart ckan` แล้ว Ctrl+F5, เช็ค `docker compose logs ckan --tail 50` |
| สีบางส่วนไม่เปลี่ยน | ใช้ F12 → Inspect ดูชื่อ class จริงแล้วแก้ใน Custom CSS |
| ปุ่ม Upload โลโก้ไม่ขึ้น | ใส่ URL ของรูปลงในช่อง Site Tag Logo แทน |

### ตรวจสอบผลรวม (11 รายการ)

- [ ] 1. ชื่อไซต์เป็น Ckan-weeb
- [ ] 2. โลโก้ใหม่แสดงที่หัวเว็บ
- [ ] 3. ธีมสีเป็นม่วง-ชมพู
- [ ] 4. หน้าหลักปรับด้วย CSS แล้ว
- [ ] 5. ข้อความเกริ่นนำเปลี่ยนแล้ว
- [ ] 6. หน้า About เปลี่ยนแล้ว
- [ ] 7. ที่อยู่ เบอร์โทร อีเมลใน Footer เปลี่ยนแล้ว
- [ ] 8. ลิงก์ใน Footer ใช้งานได้
- [ ] 9. แบนเนอร์ 420 x 220 แสดงแทนรูปเทา
- [ ] 10. พื้นหลังเปลี่ยนแล้ว
- [ ] 11. ไอคอนแท็บเบราว์เซอร์เปลี่ยนแล้ว

**หลักฐานที่ควรแคป:** หน้าแรก, หน้า About, Footer, หน้า `/ckan-admin/config`, แท็บเบราว์เซอร์ที่เห็นไอคอน
