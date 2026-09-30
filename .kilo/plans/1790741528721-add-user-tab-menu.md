# Plan: Tambah Tab Menu + Fitur Tambah User Baru

## Konteks
- File: `index.html` (single-page app, vanilla JS + Tailwind CSS)
- Endpoint baru: `POST /api/users` — body `{ "username": "..." }` — untuk membuat user baru
- UI perlu tab menu untuk berpindah antara halaman "Simulator PFP" (existing) dan "Tambah User" (baru)

## Desain

### Layout Baru
Struktur berubah dari langsung card → menjadi **card dengan tab navigation di atas konten utama (right panel)**.

```
┌──────────────────────────────────────────┐
│  [Sidebar: PFP Preview]  │  Tab Bar:    │
│  - Avatar                 │  [PFP] [User]│
│  - Username               │──────────────│
│  - Status                 │              │
│                           │  Tab Content │
│                           │  (switched)  │
└──────────────────────────────────────────┘
```

- **Sidebar tetap** — tidak berubah, selalu visible di kedua tab
- **Tab bar** ditempatkan di atas area konten right panel (menggantikan heading statis)
- 2 tab: **"Simulator PFP"** (default active) dan **"Tambah User"**
- Styling: Shadcn-style underline tabs (border-bottom aktif = primary color, inactive = muted)

### Tab 1: "Simulator PFP" (existing)
Konten yang sudah ada sekarang (config form + opsi perubahan gambar). Tidak ada perubahan.

### Tab 2: "Tambah User"
Konten baru:
- **Heading**: "Tambah User Baru" + subtitle "Buat user baru di database Neon"
- **Input**: "Base URL API" — **shared** dengan tab PFP (menggunakan `#apiUrl` yang sama, sudah ada di atas). Karena `apiUrl` ada di tab PFP, tab ini perlu membaca nilainya langsung via `getBaseApiUrl()` tanpa input sendiri (atau kita pindahkan `apiUrl` ke sidebar agar shared). **Keputusan**: cukup baca dari `#apiUrl` yang ada — user harus set URL dulu di tab PFP, tapi ini acceptable karena simple app.
- **Input**: "Username" — text input (`id="newUsername"`)
- **Button**: "Tambah User" — memanggil `POST /api/users` dengan body `{ "username": value }`
- **Response area**: Menampilkan ID user yang baru dibuat (dari response API) agar user bisa langsung copy ID-nya ke field "ID User" di tab PFP
- Semua aktivitas network juga di-log ke Network Inspector

## Langkah Implementasi

### 1. Tambahkan tab bar di right panel
- Di dalam `.flex-1` div (line 56), **sebelum** konten yang ada, tambahkan tab bar dengan 2 tombol
- Tab bar: `<div class="flex border-b border-border mb-6">` berisi 2 button tab
- Active tab styling: `border-b-2 border-primary text-foreground font-semibold`
- Inactive tab styling: `text-muted-foreground hover:text-foreground`

### 2. Wrap konten existing dalam tab panel
- Bungkus semua konten existing (line 57-100) dalam `<div id="tabPfp">` 
- Buat `<div id="tabAddUser" class="hidden">` untuk konten tab baru

### 3. Buat konten tab "Tambah User"
Di dalam `#tabAddUser`:
```html
<div class="mb-6">
  <h2>Tambah User Baru</h2>
  <p>Buat user baru di database Neon.</p>
</div>

<div class="space-y-4">
  <div class="space-y-1.5">
    <label>Username</label>
    <input type="text" id="newUsername" placeholder="contoh: ridwan">
  </div>
  
  <button onclick="createNewUser()">Tambah User</button>
</div>

<!-- Response info -->
<div id="createUserResult" class="hidden mt-6 p-4 bg-secondary rounded-lg">
  <p class="text-sm text-muted-foreground">User berhasil dibuat:</p>
  <p class="font-mono text-sm">ID: <span id="newUserId"></span></p>
  <p class="font-mono text-sm">Username: <span id="newUserName"></span></p>
  <button onclick="useNewUserId()">Gunakan ID ini di Simulator PFP</button>
</div>
```

### 4. Tambahkan JavaScript functions

**`switchTab(tabName)`** — Toggle visibility antara `#tabPfp` dan `#tabAddUser`, update active tab styling.

**`createNewUser()`**:
```js
async function createNewUser() {
  const apiUrl = getBaseApiUrl();
  const username = document.getElementById('newUsername').value.trim();
  
  if (!apiUrl) return alert("Set Base URL API dulu di tab Simulator PFP!");
  if (!username) return alert("Masukkan username!");
  
  openTerminal();
  logMessage(`\n=== MEMBUAT USER BARU ===`);
  logMessage(`[API REQUEST] POST ${apiUrl}/api/users`, "warn");
  
  try {
    const response = await fetch(`${apiUrl}/api/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username })
    });
    const result = await response.json();
    if (!response.ok) throw new Error(result.message || 'Error dari Backend');
    
    logMessage(`[API RESPONSE] 201 Created - User Berhasil Dibuat!`);
    logMessage(`[DB RECORD] ${JSON.stringify(result.data || result)}`);
    
    // Show result panel with new user info
    // Display ID and username from response
  } catch (err) {
    logMessage(`[ERROR] ${err.message}`, "error");
    alert("Gagal membuat user. Cek API Backend kamu!");
  }
}
```

**`useNewUserId()`** — Copy ID user baru ke `#userId` input, switch ke tab PFP.

### 5. Pindahkan Base URL API ke area shared (opsional, recommended)
- Agar `apiUrl` input bisa diakses dari kedua tab, pindahkan ke **sidebar** di bawah status badge, atau biarkan tetap di tab PFP dan cukup baca value-nya via `getBaseApiUrl()`.
- **Keputusan**: Pindahkan `apiUrl` input ke atas tab bar (tetap di right panel tapi sebelum tabs) agar terlihat di kedua tab.

## Validation
- Buka `index.html` di browser
- Tab "Simulator PFP" harus menampilkan konten existing persis sama
- Tab "Tambah User" menampilkan form username + tombol
- Klik "Tambah User" → POST ke `/api/users` → log di Network Inspector → tampilkan result
- Klik "Gunakan ID ini" → ID otomatis pindah ke field userId + switch ke tab PFP
- Tab switching smooth, no flicker

## Risks
- `apiUrl` input ada di tab PFP — user mungkin lupa set sebelum pindah ke tab "Tambah User". Mitigasi: pindahkan ke area shared di atas tabs.
- Response body dari `POST /api/users` tidak diketahui pasti (asumsi: mengembalikan `{ data: { id, username } }` atau `{ id, username }`). Kode harus handle kedua format.
