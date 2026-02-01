Bootcamp Session 2
Materi Video / Rekaman
https://www.youtube.com/watch?v=47BLJ3EPNAw

Materi Teks
Assalamu’alaikum temen-temen

Saya ucapkan selamat kepada temen-temen yang ada disini berarti kalian termasuk orang-orang yang lolos dari 1300an orang.

Ya, jadi kita kemarin udah lihat betapa gampangnya kita nge-develop di Golang, betapa kecilnya memori yang dibutuhkan di Golang, dan betapa gampangnya kita deploy gratis lagi, di Railway ataupun Zeabur, atau ada yang di leapcell kemarin ya. 

Nah, teman-teman, aplikasi kita kemarin itu enggak sehat. Karena ada tiga hal, yaitu 

kita taruh semua kode di main.go. Jadi, kalau nambah fitur, nambah apapun itu, code bakal susah nantinya ke depannya ya. 

Kemudian, kita cuma simpan data itu di RAM. Jadi, kalau kita close server-nya, server akan mati, akan hilang datanya. 

Yang ketiga, config kita langsung tulis di kode. Jadi, kalau misalkan mau ganti database, atau ganti, mau ganti port, itu harus bongkar lagi kodenya, jadi enggak fleksibel kan.

Sekarang kita bahas satu persatu

Kode Yang Berantakan
Ada beberapa standar yang biasa dipakai di golang untuk project structure

Standard Go Project Layout
https://github.com/golang-standards/project-layout → referensi paling populer

Martin Fowler – Layered Architecture
https://martinfowler.com/bliki/LayeredArchitecture.html → memisahkan urusan teknis dan bisnis

Clean Architecture – Uncle Bob
https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html → Dia ini kayak 'Kakek'-nya Clean Code di dunia programmer

Kita akan implementasikan Layered Architecture, dimana setiap bagian / folder punya tanggung jawab yang jelas.

Handler
Menerima Request dan response

Service
Logic Kode kita

Repository
Data buat logic 

Model
Tempat buat definisi bentuk data 

Cara bacanya jadi gampang

Misal ada error di database → Repository

Misal ada error logic nya → Service

Misal ada error request nya → Handler

Analogi Restoran




Request / Pesanan masuk dari pelanggan akan diterima oleh Pelayan, Pelayan disini ngecek valid nggak nih pesanannya, ada nggak dimenunya. kalau valid lanjut kedapur

Di dapur pelayan bilang ke Koki: “nih ada pesanan”. Koki: “Oke dikerjakan”

Koki lihat pesanannya butuh beberapa bahan, bilang lah ke Anak Gudang, lalu Anak Gudang kasih bahan-bahannya, Koki yang masak, balikin ke Pelayan buat dikasih ke pelanggan yang memesan

Dependency Injection
Sebelum restoran kita buka, seorang Manager akan meminta untuk Pelayan, Koki, Anak Gudang itu saling mengenal. 

Pelayan A → kalau ada Pesanan Nasi Goreng kamu hubungi Koki 1 ya

Koki 1 → kalau ada Pesanan Nasi Goreng, kamu bisa hubungi Anak Gudang I ya

Inisiasi inilah yang kita sebut Injection, dimana kita akan lakukan di main.go sebelum request itu datang, atau saat pertama kali server running (Dependency Injection).

Database
Gambaran database adalah seperti excel sheet





Nomor 1 adalah database

Nomor 2 dan 3 kita sebut sebagai table

Nomor 4, 5, 6 adalah kolom

Kita akan pakai database tanpa setup-setup yang susah, yaitu pakai Supabase

Buka Supabase.com, kemudian login ke dashboard nya





Bikin Organisasi





isi data organisasi, dan create





Create project





Isi nama dan generate password buat db (Simpan ini nanti kita pakai)





Masuk ke Table Editor, kita akan bikin table kita





Klik New table





Isi table name nya, kita buat table product dulu

Kemudian isi semua kolom yang kita butuhkan

name varchar
price int4
stock int4
Terakhir kita save







Simpan credential buat nanti koneksi dari aplikasi kita, caranya klik connect





Pilih transaction pooler





Copy db url nya, jangan lupa ganti placeholder PASSWORD dengan password temen-temen





Kalau lupa password bisa generate ulang, caranya lewat Database → Setting → Reset Database Password





Config
config itu adalah setting yang bisa kita sesuaikan tanpa mengubah mesin atau logic. misalkan saya punya vscode ak setting gelap mungkin dilaptop saya yang lain saya setting jadi putih, sama-sama vscode tapi dibedain setting nya. 

Nah kali ini kita akan pakai viper dari golang https://github.com/spf13/viper, karena ini library config di golang yang sangat populer, kenapa. karena sangat lengkap. 

Kita mulai dengan ganti port kita ke setting / config, karena ketika deploy misalkan kita mau beda port, tinggal ganti confignya

Caranya:

Kita install dulu Viper dengan cara go get https://github.com/spf13/viper

kemudian di main.go kita define dulu config di dalam tipe data, kita namai Config isinya adalah PORT

type Config struct {
	Port    string `mapstructure:"PORT"`
}
disini kita define juga tag mapstructure:"PORT”, yang artinya kalau config ini diambil dengan viper dia perlu PORT 

kita butuh bikin .env yang bakal dipakai viper buat define confignya

PORT=8080
  4. Karena kita udah define .env, kita butuh baca .env dengan viper caranya

viper.AutomaticEnv()
viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

if _, err := os.Stat(".env"); err == nil {
	viper.SetConfigFile(".env")
	_ = viper.ReadInConfig()
}
Selanjutnya kita bisa ambil config kita dari viper dengan cara kita define dulu variablenya dan kita assign ke environment nya

config := Config{
 	Port: viper.GetString("PORT"),
}
Kita implement config ke dalam server load kita dengan edit line ini

addr := "0.0.0.0:" + config.Port
fmt.Println("Server running di", addr)

err = http.ListenAndServe(addr, nil)
if err != nil {
	fmt.Println("gagal running server", err)
}
Implement Database
database/database.go

package database

import (
	"database/sql"
	"log"

	_ "github.com/lib/pq"
)

func InitDB(connectionString string) (*sql.DB, error) {
	// Open database
	db, err := sql.Open("postgres", connectionString)
	if err != nil {
		return nil, err
	}

	// Test connection
	err = db.Ping()
	if err != nil {
		return nil, err
	}

	// Set connection pool settings (optional tapi recommended)
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)

	log.Println("Database connected successfully")
	return db, nil
}

main.go

// ubah Config
type Config struct {
	Port   string `mapstructure:"PORT"`
	DBConn string `mapstructure:"DB_CONN"`
}

config := Config{
		Port:   viper.GetString("PORT"),
		DBConn: viper.GetString("DB_CONN"),
}

// Setup database
	db, err := database.InitDB(config.DBConn)
	if err != nil {
		log.Fatal("Failed to initialize database:", err)
	}
	defer db.Close()
Pindah Model
model/product.go

package models

type Product struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Price int    `json:"price"`
	Stock int    `json:"stock"`
}
Pindah Handler
handlers/product_handler.go

package handlers

import (
	"encoding/json"
	"kasir-api/models"
	"kasir-api/services"
	"net/http"
	"strconv"
	"strings"
)

type ProductHandler struct {
	service *services.ProductService
}

func NewProductHandler(service *services.ProductService) *ProductHandler {
	return &ProductHandler{service: service}
}

// HandleProducts - GET /api/produk
func (h *ProductHandler) HandleProducts(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		h.GetAll(w, r)
	case http.MethodPost:
		h.Create(w, r)
	default:
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}

func (h *ProductHandler) GetAll(w http.ResponseWriter, r *http.Request) {
	products, err := h.service.GetAll()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(products)
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
	var product models.Product
	err := json.NewDecoder(r.Body).Decode(&product)
	if err != nil {
		http.Error(w, "Invalid request body", http.StatusBadRequest)
		return
	}

	err = h.service.Create(&product)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(product)
}

// HandleProductByID - GET/PUT/DELETE /api/produk/{id}
func (h *ProductHandler) HandleProductByID(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		h.GetByID(w, r)
	case http.MethodPut:
		h.Update(w, r)
	case http.MethodDelete:
		h.Delete(w, r)
	default:
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}

// GetByID - GET /api/produk/{id}
func (h *ProductHandler) GetByID(w http.ResponseWriter, r *http.Request) {
	idStr := strings.TrimPrefix(r.URL.Path, "/api/produk/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "Invalid product ID", http.StatusBadRequest)
		return
	}

	product, err := h.service.GetByID(id)
	if err != nil {
		http.Error(w, err.Error(), http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(product)
}

func (h *ProductHandler) Update(w http.ResponseWriter, r *http.Request) {
	idStr := strings.TrimPrefix(r.URL.Path, "/api/produk/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "Invalid product ID", http.StatusBadRequest)
		return
	}

	var product models.Product
	err = json.NewDecoder(r.Body).Decode(&product)
	if err != nil {
		http.Error(w, "Invalid request body", http.StatusBadRequest)
		return
	}

	product.ID = id
	err = h.service.Update(&product)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(product)
}

// Delete - DELETE /api/produk/{id}
func (h *ProductHandler) Delete(w http.ResponseWriter, r *http.Request) {
	idStr := strings.TrimPrefix(r.URL.Path, "/api/produk/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "Invalid product ID", http.StatusBadRequest)
		return
	}

	err = h.service.Delete(id)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Product deleted successfully",
	})
}


Define route di main.go

// Setup routes
http.HandleFunc("/api/produk", productHandler.HandleProducts)
http.HandleFunc("/api/produk/", productHandler.HandleProductByID)
Pindah Service
sevices/product_service.go

package services

import (
	"kasir-api/models"
	"kasir-api/repositories"
)

type ProductService struct {
	repo *repositories.ProductRepository
}

func NewProductService(repo *repositories.ProductRepository) *ProductService {
	return &ProductService{repo: repo}
}

func (s *ProductService) GetAll() ([]models.Product, error) {
	return s.repo.GetAll()
}

func (s *ProductService) Create(data *models.Product) error {
	return s.repo.Create(data)
}

func (s *ProductService) GetByID(id int) (*models.Product, error) {
	return s.repo.GetByID(id)
}

func (s *ProductService) Update(product *models.Product) error {
	return s.repo.Update(product)
}

func (s *ProductService) Delete(id int) error {
	return s.repo.Delete(id)
}

Pindah Repository
repositories/product_repository.go

package repositories

import (
	"database/sql"
	"errors"
	"kasir-api/models"
)

type ProductRepository struct {
	db *sql.DB
}

func NewProductRepository(db *sql.DB) *ProductRepository {
	return &ProductRepository{db: db}
}

func (repo *ProductRepository) GetAll() ([]models.Product, error) {
	query := "SELECT id, name, price, stock FROM products"
	rows, err := repo.db.Query(query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	products := make([]models.Product, 0)
	for rows.Next() {
		var p models.Product
		err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
		if err != nil {
			return nil, err
		}
		products = append(products, p)
	}

	return products, nil
}

func (repo *ProductRepository) Create(product *models.Product) error {
	query := "INSERT INTO products (name, price, stock) VALUES ($1, $2, $3) RETURNING id"
	err := repo.db.QueryRow(query, product.Name, product.Price, product.Stock).Scan(&product.ID)
	return err
}

// GetByID - ambil produk by ID
func (repo *ProductRepository) GetByID(id int) (*models.Product, error) {
	query := "SELECT id, name, price, stock FROM products WHERE id = $1"

	var p models.Product
	err := repo.db.QueryRow(query, id).Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
	if err == sql.ErrNoRows {
		return nil, errors.New("produk tidak ditemukan")
	}
	if err != nil {
		return nil, err
	}

	return &p, nil
}

func (repo *ProductRepository) Update(product *models.Product) error {
	query := "UPDATE products SET name = $1, price = $2, stock = $3 WHERE id = $4"
	result, err := repo.db.Exec(query, product.Name, product.Price, product.Stock, product.ID)
	if err != nil {
		return err
	}

	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}

	if rows == 0 {
		return errors.New("produk tidak ditemukan")
	}

	return nil
}

func (repo *ProductRepository) Delete(id int) error {
	query := "DELETE FROM products WHERE id = $1"
	result, err := repo.db.Exec(query, id)
	if err != nil {
		return err
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}

	if rows == 0 {
		return errors.New("produk tidak ditemukan")
	}

	return err
}

Dependency Injection
main.go

productRepo := repositories.NewProductRepository(db)
productService := services.NewProductService(productRepo)
productHandler := handlers.NewProductHandler(productService)
Deploy ke railway
Sebelum push code ke github kita setting variable (env) di railway, tambahkan 

PORT=8080
DB_CONN=postgresql://postgres.[PROJECT_ID]:[YOUR-PASSWORD]@aws-1-ap-south-1.pooler.supabase.com:6543/postgres




Push code ke github

git add .
git commit -m "layered arch, env, database"
git push
Railway Akan Otomatis deploy code baru kita. 

Task Session 2
Pindah categories temen-temen ke layered architecture

Challange (Optional): Explore Join, tambah category_id ke table products, setiap product mempunyai kategory, dan ketika Get Detail return category.name dari product