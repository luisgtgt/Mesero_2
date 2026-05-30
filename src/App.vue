
<template>
  <div class="app-container">
    <header class="topbar">
      <div>
        <h1>{{ company.name }}</h1>
        <p>Sistema de pedidos y facturación</p>
      </div>

      <div class="top-actions">
        <button class="cart-btn" @click="toggleCart">
          🛒 Carrito ({{ cart.length }})
        </button>

        <button class="table-btn" @click="showTablePanel = true">
          🍽️ Mesas
        </button>

        <button class="company-btn" @click="showCompanyPanel = true">
          🏢 Empresa
        </button>

        <button class="invoice-btn" @click="showInvoices = true">
          📄 Facturas
        </button>
      </div>
    </header>

    <section class="filters-section">
      <div class="search-box">
        <input
          type="text"
          placeholder="Buscar producto..."
          v-model="search"
        />
      </div>

      <div class="categories">
        <button
          v-for="cat in categories"
          :key="cat"
          :class="selectedCategory === cat ? 'active-category' : ''"
          @click="selectedCategory = cat"
        >
          {{ cat }}
        </button>
      </div>
    </section>

    <button class="floating-add" @click="openAddModal">
      + Nuevo Producto
    </button>

    <section class="products-grid">
      <div
        class="product-card"
        v-for="product in filteredProducts()"
        :key="product.id"
      >
        <img :src="product.image" :alt="product.name" />

        <div class="product-info">
          <h3>{{ product.name }}</h3>
          <p class="category">{{ product.category }}</p>
          <p class="price">${{ formatPrice(product.price) }}</p>
          <p class="code">ID: {{ product.id }}</p>
        </div>

        <div class="card-actions">
          <button class="minus" @click="removeFromCart(product)">
            -
          </button>

          <button class="plus" @click="addToCart(product)">
            +
          </button>
        </div>

        <div class="edit-actions">
          <button @click="editProduct(product)">Editar</button>
          <button class="danger" @click="deleteProduct(product.id)">
            Eliminar
          </button>
        </div>
      </div>
    </section>

    <!-- MODAL PRODUCTO -->
    <div v-if="showProductModal" class="modal-overlay">
      <div class="modal">
        <h2>
          {{ editingProduct ? 'Editar Producto' : 'Nuevo Producto' }}
        </h2>

        <div class="form-group">
          <label>Nombre</label>
          <input type="text" v-model="form.name" />
        </div>

        <div class="form-group">
          <label>Precio</label>
          <input type="text" v-model="form.price" />
        </div>

        <div class="form-group">
          <label>ID</label>
          <input type="text" v-model="form.id" />
        </div>

        <div class="form-group">
          <label>Categoría</label>
          <select v-model="form.category">
            <option>Hamburguesas</option>
            <option>Bebidas</option>
            <option>Salchipapa</option>
            <option>Otros</option>
          </select>
        </div>

        <div class="form-group">
          <label>Imagen URL</label>
          <input type="text" v-model="form.image" />
        </div>

        <div class="modal-actions">
          <button @click="saveProduct">Guardar</button>
          <button class="danger" @click="closeProductModal">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <!-- CARRITO -->
    <div v-if="showCart" class="cart-panel">
      <div class="cart-header">
        <h2>Pedido Actual</h2>
        <button @click="showCart = false">✖</button>
      </div>

      <div v-if="cart.length === 0" class="empty-cart">
        No hay productos en el carrito
      </div>

      <div class="cart-items">
        <div class="cart-item" v-for="item in cart" :key="item.id">
          <img :src="item.image" />

          <div class="cart-info">
            <h4>{{ item.name }}</h4>
            <p>Cantidad: {{ item.quantity }}</p>
            <p>${{ formatPrice(item.price * item.quantity) }}</p>
          </div>

          <div class="cart-actions-buttons">
            <button @click="addToCart(item)">+</button>
            <button @click="removeFromCart(item)">-</button>
            <button class="danger" @click="removeItem(item.id)">
              Eliminar
            </button>
          </div>
        </div>
      </div>

      <div class="cart-footer">
        <h3>Total: ${{ formatPrice(cartTotal()) }}</h3>

        <button class="checkout-btn" @click="openCheckout">
          Facturar Pedido
        </button>
      </div>
    </div>

    <!-- CHECKOUT -->
    <div v-if="showCheckout" class="modal-overlay">
      <div class="modal checkout-modal">
        <h2>Generar Factura</h2>

        <div class="form-group">
          <label>Cliente</label>
          <input type="text" v-model="invoiceForm.client" />
        </div>

        <div class="form-group">
          <label>Documento</label>
          <input type="text" v-model="invoiceForm.document" />
        </div>

        <div class="form-group">
          <label>Mesa</label>
          <select v-model="invoiceForm.table">
            <option
              v-for="table in tables"
              :key="table.id"
              :value="table.name"
            >
              {{ table.name }}
            </option>
          </select>
        </div>

        <div class="modal-actions">
          <button @click="generateInvoice">
            Generar Factura
          </button>

          <button class="danger" @click="showCheckout = false">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <!-- FACTURAS -->
    <div v-if="showInvoices" class="side-panel">
      <div class="panel-header">
        <h2>Facturas Generadas</h2>
        <button @click="showInvoices = false">✖</button>
      </div>

      <div class="invoice-list">
        <div
          class="invoice-card"
          v-for="invoice in invoices"
          :key="invoice.number"
        >
          <h3>Factura #{{ invoice.number }}</h3>
          <p>Cliente: {{ invoice.client }}</p>
          <p>Mesa: {{ invoice.table }}</p>
          <p>Total: ${{ formatPrice(invoice.total) }}</p>

          <button @click="downloadInvoice(invoice)">
            Descargar PDF
          </button>
        </div>
      </div>
    </div>

    <!-- MESAS -->
    <div v-if="showTablePanel" class="side-panel">
      <div class="panel-header">
        <h2>Administrar Mesas</h2>
        <button @click="showTablePanel = false">✖</button>
      </div>

      <div class="table-form">
        <input type="text" v-model="newTable" placeholder="Nueva mesa" />
        <button @click="addTable">Agregar</button>
      </div>

      <div class="table-list">
        <div class="table-item" v-for="table in tables" :key="table.id">
          <input type="text" v-model="table.name" />

          <button @click="saveTables">Guardar</button>

          <button class="danger" @click="deleteTable(table.id)">
            Eliminar
          </button>
        </div>
      </div>
    </div>

    <!-- EMPRESA -->
    <div v-if="showCompanyPanel" class="side-panel">
      <div class="panel-header">
        <h2>Datos Empresa</h2>
        <button @click="showCompanyPanel = false">✖</button>
      </div>

      <div class="form-group">
        <label>Nombre</label>
        <input type="text" v-model="company.name" />
      </div>

      <div class="form-group">
        <label>NIT</label>
        <input type="text" v-model="company.nit" />
      </div>

      <div class="form-group">
        <label>Dirección</label>
        <input type="text" v-model="company.address" />
      </div>

      <div class="form-group">
        <label>Teléfono</label>
        <input type="text" v-model="company.phone" />
      </div>

      <button @click="saveCompany">Guardar Datos</button>
    </div>

    <!-- NOTIFICACIONES -->
    <transition name="fade">
      <div v-if="notification.show" class="notification">
        {{ notification.message }}
      </div>
    </transition>

    <!-- SPINNER -->
    <div v-if="loading" class="loading-screen">
      <div class="spinner"></div>
      <p>Procesando...</p>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      search: '',
      selectedCategory: 'Todos',
      showProductModal: false,
      showCart: false,
      showCheckout: false,
      showInvoices: false,
      showTablePanel: false,
      showCompanyPanel: false,
      editingProduct: false,
      loading: false,
      newTable: '',

      notification: {
        show: false,
        message: ''
      },

      invoiceForm: {
        client: '',
        document: '',
        table: ''
      },

      company: {
        name: 'Burger House',
        nit: '900123456',
        address: 'Bogotá - Colombia',
        phone: '3000000000'
      },

      categories: [
        'Todos',
        'Hamburguesas',
        'Bebidas',
        'Salchipapa',
        'Otros'
      ],

      form: {
        id: '',
        name: '',
        price: '',
        image: '',
        category: 'Hamburguesas'
      },

      cart: [],
      invoices: [],

      tables: [
        { id: 1, name: 'Mesa 1' },
        { id: 2, name: 'Mesa 2' },
        { id: 3, name: 'Mesa 3' }
      ],

      products: []
    }
  },

  mounted() {
    this.loadProducts()
    this.loadInvoices()
    this.loadTables()
    this.loadCompany()
  },

  methods: {
    loadProducts() {
      const saved = localStorage.getItem('products')

      if (saved) {
        this.products = JSON.parse(saved)
      } else {
        this.products = this.defaultProducts()
        localStorage.setItem('products', JSON.stringify(this.products))
      }
    },

    defaultProducts() {
      return [
        {
          id: 'H001',
          name: 'Hamburguesa Clásica',
          price: 18000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1568901346375-23c9450c58cd'
        },
        {
          id: 'H002',
          name: 'Hamburguesa Doble',
          price: 24000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1550547660-d9450f859349'
        },
        {
          id: 'H003',
          name: 'Hamburguesa BBQ',
          price: 26000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1571091718767-18b5b1457add'
        },
        {
          id: 'H004',
          name: 'Hamburguesa Ranchera',
          price: 25000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1550317138-10000687a72b'
        },
        {
          id: 'H005',
          name: 'Hamburguesa Mexicana',
          price: 28000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1586190848861-99aa4a171e90'
        },
        {
          id: 'B001',
          name: 'Coca Cola',
          price: 5000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1581636625402-29b2a704ef13'
        },
        {
          id: 'B002',
          name: 'Pepsi',
          price: 5000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1629203851122-3726ecdf080e'
        },
        {
          id: 'B003',
          name: 'Jugo Natural',
          price: 7000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1623065422902-30a2d299bbe4'
        },
        {
          id: 'B004',
          name: 'Limonada',
          price: 8000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1513558161293-cdaf765ed2fd'
        },
        {
          id: 'B005',
          name: 'Malteada',
          price: 12000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1579954115545-a95591f28bfc'
        },
        {
          id: 'S001',
          name: 'Salchipapa Personal',
          price: 17000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1630383249896-424e482df921'
        },
        {
          id: 'S002',
          name: 'Salchipapa Especial',
          price: 24000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1518013431117-eb1465fa5752'
        },
        {
          id: 'S003',
          name: 'Salchipapa Mixta',
          price: 28000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1562967914-608f82629710'
        },
        {
          id: 'S004',
          name: 'Salchipapa BBQ',
          price: 29000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1606755962773-d324e0a13086'
        },
        {
          id: 'O001',
          name: 'Papas Francesas',
          price: 9000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1576107232684-1279f390859f'
        },
        {
          id: 'O002',
          name: 'Nuggets',
          price: 15000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1562967916-eb82221dfb92'
        },
        {
          id: 'O003',
          name: 'Perro Caliente',
          price: 16000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1612392062798-56f5b1f0d7a2'
        },
        {
          id: 'O004',
          name: 'Arepa Burger',
          price: 20000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1594212699903-ec8a3eca50f5'
        },
        {
          id: 'O005',
          name: 'Combo Familiar',
          price: 45000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1565299624946-b28f40a0ae38'
        },
        {
          id: 'H006',
          name: 'Hamburguesa Triple',
          price: 32000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1520072959219-c595dc870360'
        },
        {
          id: 'H007',
          name: 'Hamburguesa Pollo',
          price: 22000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1525059696034-4967a8e1dca2'
        },
        {
          id: 'H008',
          name: 'Hamburguesa Crispy',
          price: 23000,
          category: 'Hamburguesas',
          image: 'https://images.unsplash.com/photo-1600891964599-f61ba0e24092'
        },
        {
          id: 'B006',
          name: 'Agua',
          price: 3000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1564419320461-6870880221ad'
        },
        {
          id: 'B007',
          name: 'Té Helado',
          price: 6000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1499636136210-6f4ee915583e'
        },
        {
          id: 'B008',
          name: 'Café',
          price: 4000,
          category: 'Bebidas',
          image: 'https://images.unsplash.com/photo-1495474472287-4d71bcdd2085'
        },
        {
          id: 'S005',
          name: 'Salchipapa Costeña',
          price: 26000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1504674900247-0877df9cc836'
        },
        {
          id: 'S006',
          name: 'Salchipapa Suprema',
          price: 31000,
          category: 'Salchipapa',
          image: 'https://images.unsplash.com/photo-1565299507177-b0ac66763828'
        },
        {
          id: 'O006',
          name: 'Aros de Cebolla',
          price: 12000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1639024471283-03518883512d'
        },
        {
          id: 'O007',
          name: 'Pizza Personal',
          price: 22000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1513104890138-7c749659a591'
        },
        {
          id: 'O008',
          name: 'Wrap Pollo',
          price: 18000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1626700051175-6818013e1d4f'
        },
        {
          id: 'O009',
          name: 'Brownie',
          price: 9000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1606313564200-e75d5e30476c'
        },
        {
          id: 'O010',
          name: 'Helado',
          price: 8000,
          category: 'Otros',
          image: 'https://images.unsplash.com/photo-1563805042-7684c019e1cb'
        }
      ]
    },

    filteredProducts() {
      let list = this.products

      if (this.selectedCategory !== 'Todos') {
        list = list.filter(
          p => p.category === this.selectedCategory
        )
      }

      if (this.search.trim() !== '') {
        list = list.filter(p =>
          p.name.toLowerCase().includes(this.search.toLowerCase())
        )
      }

      return list
    },

    openAddModal() {
      this.editingProduct = false

      this.form = {
        id: '',
        name: '',
        price: '',
        image: '',
        category: 'Hamburguesas'
      }

      this.showProductModal = true
    },

    closeProductModal() {
      this.showProductModal = false
    },

    saveProduct() {
      if (this.form.name.trim() === '') {
        this.showNotification('El nombre es obligatorio')
        return
      }

      if (this.form.price.trim() === '') {
        this.showNotification('El precio es obligatorio')
        return
      }

      if (isNaN(this.form.price)) {
        this.showNotification('El precio debe ser numérico')
        return
      }

      if (this.form.id.trim() === '') {
        this.showNotification('El ID es obligatorio')
        return
      }

      if (this.form.image.trim() === '') {
        this.showNotification('La imagen es obligatoria')
        return
      }

      if (this.editingProduct) {
        const index = this.products.findIndex(
          p => p.id === this.form.id
        )

        this.products[index] = {
          ...this.form,
          price: Number(this.form.price)
        }

        this.showNotification('Producto actualizado')
      } else {
        this.products.push({
          ...this.form,
          price: Number(this.form.price)
        })

        this.showNotification('Producto agregado')
      }

      localStorage.setItem('products', JSON.stringify(this.products))

      this.closeProductModal()
    },

    editProduct(product) {
      this.editingProduct = true
      this.form = { ...product }
      this.showProductModal = true
    },

    deleteProduct(id) {
      this.products = this.products.filter(p => p.id !== id)

      localStorage.setItem('products', JSON.stringify(this.products))

      this.showNotification('Producto eliminado')
    },

    addToCart(product) {
      const existing = this.cart.find(p => p.id === product.id)

      if (existing) {
        existing.quantity++
      } else {
        this.cart.push({
          ...product,
          quantity: 1
        })
      }

      this.showNotification('Producto agregado al carrito')
    },

    removeFromCart(product) {
      const existing = this.cart.find(p => p.id === product.id)

      if (!existing) {
        this.showNotification('Ese producto no está en el carrito')
        return
      }

      existing.quantity--

      if (existing.quantity <= 0) {
        this.cart = this.cart.filter(p => p.id !== product.id)
      }
    },

    removeItem(id) {
      this.cart = this.cart.filter(p => p.id !== id)
      this.showNotification('Producto eliminado del carrito')
    },

    toggleCart() {
      this.showCart = !this.showCart
    },

    cartTotal() {
      let total = 0

      this.cart.forEach(item => {
        total += item.price * item.quantity
      })

      return total
    },

    openCheckout() {
      if (this.cart.length === 0) {
        this.showNotification('El carrito está vacío')
        return
      }

      this.showCheckout = true
    },

    generateInvoice() {
      if (this.invoiceForm.client.trim() === '') {
        this.showNotification('Ingrese el nombre del cliente')
        return
      }

      if (this.invoiceForm.document.trim() === '') {
        this.showNotification('Ingrese el documento')
        return
      }

      if (this.invoiceForm.table.trim() === '') {
        this.showNotification('Seleccione una mesa')
        return
      }

      this.loading = true

      setTimeout(() => {
        const invoice = {
          number: Date.now(),
          client: this.invoiceForm.client,
          document: this.invoiceForm.document,
          table: this.invoiceForm.table,
          items: [...this.cart],
          total: this.cartTotal(),
          date: new Date().toLocaleString()
        }

        this.invoices.push(invoice)

        localStorage.setItem(
          'invoices',
          JSON.stringify(this.invoices)
        )

        this.cart = []

        this.invoiceForm = {
          client: '',
          document: '',
          table: ''
        }

        this.loading = false
        this.showCheckout = false

        this.showNotification('Factura generada exitosamente')
      }, 2000)
    },

    downloadInvoice(invoice) {
      const content = `
        <html>
          <head>
            <title>Factura</title>
            <style>
              body{
                font-family: Arial;
                padding:40px;
              }

              .header{
                text-align:center;
                margin-bottom:30px;
              }

              table{
                width:100%;
                border-collapse: collapse;
              }

              th, td{
                border:1px solid #ddd;
                padding:12px;
                text-align:left;
              }

              th{
                background:#111;
                color:white;
              }

              .footer{
                margin-top:40px;
                text-align:center;
              }
            </style>
          </head>

          <body>
            <div class="header">
              <h1>${this.company.name}</h1>
              <p>NIT: ${this.company.nit}</p>
              <p>${this.company.address}</p>
              <p>${this.company.phone}</p>
            </div>

            <h2>Factura #${invoice.number}</h2>

            <p><strong>Cliente:</strong> ${invoice.client}</p>
            <p><strong>Documento:</strong> ${invoice.document}</p>
            <p><strong>Mesa:</strong> ${invoice.table}</p>
            <p><strong>Fecha:</strong> ${invoice.date}</p>

            <table>
              <thead>
                <tr>
                  <th>Producto</th>
                  <th>Cantidad</th>
                  <th>Precio</th>
                  <th>Subtotal</th>
                </tr>
              </thead>

              <tbody>
                ${invoice.items.map(item => `
                  <tr>
                    <td>${item.name}</td>
                    <td>${item.quantity}</td>
                    <td>$${this.formatPrice(item.price)}</td>
                    <td>$${this.formatPrice(item.price * item.quantity)}</td>
                  </tr>
                `).join('')}
              </tbody>
            </table>

            <h2>Total: $${this.formatPrice(invoice.total)}</h2>

            <div class="footer">
              <p>Gracias por su compra</p>
              <p>${this.company.name}</p>
            </div>
          </body>
        </html>
      `

      const win = window.open('', '', 'width=900,height=1200')

      win.document.write(content)
      win.document.close()
      win.print()
    },

    loadInvoices() {
      const saved = localStorage.getItem('invoices')

      if (saved) {
        this.invoices = JSON.parse(saved)
      }
    },

    addTable() {
      if (this.newTable.trim() === '') {
        this.showNotification('Ingrese el nombre de la mesa')
        return
      }

      this.tables.push({
        id: Date.now(),
        name: this.newTable
      })

      this.newTable = ''

      this.saveTables()

      this.showNotification('Mesa agregada')
    },

    saveTables() {
      localStorage.setItem('tables', JSON.stringify(this.tables))
      this.showNotification('Mesas actualizadas')
    },

    loadTables() {
      const saved = localStorage.getItem('tables')

      if (saved) {
        this.tables = JSON.parse(saved)
      }
    },

    deleteTable(id) {
      this.tables = this.tables.filter(t => t.id !== id)
      this.saveTables()
    },

    saveCompany() {
      localStorage.setItem('company', JSON.stringify(this.company))
      this.showNotification('Datos guardados')
    },

    loadCompany() {
      const saved = localStorage.getItem('company')

      if (saved) {
        this.company = JSON.parse(saved)
      }
    },

    showNotification(message) {
      this.notification.message = message
      this.notification.show = true

      setTimeout(() => {
        this.notification.show = false
      }, 3000)
    },

    formatPrice(value) {
      return Number(value).toLocaleString('es-CO')
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: Arial, sans-serif;
}

body {
  background: #f5f5f5;
}

.app-container {
  min-height: 100vh;
  padding: 20px;
}

.topbar {
  background: #111;
  color: white;
  padding: 20px;
  border-radius: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
  flex-wrap: wrap;
}

.top-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.top-actions button,
.floating-add,
.checkout-btn,
.modal-actions button,
.invoice-card button,
.table-form button,
.table-item button,
.side-panel button {
  background: #111;
  color: white;
  border: none;
  padding: 12px 18px;
  border-radius: 12px;
  cursor: pointer;
  transition: 0.3s;
}

button:hover {
  opacity: 0.8;
}

.danger {
  background: crimson !important;
}

.filters-section {
  margin-top: 20px;
  display: flex;
  justify-content: space-between;
  gap: 20px;
  flex-wrap: wrap;
}

.search-box input {
  width: 300px;
  max-width: 100%;
  padding: 14px;
  border-radius: 12px;
  border: 1px solid #ccc;
}

.categories {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.categories button {
  padding: 10px 15px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
}

.active-category {
  background: #111;
  color: white;
}

.floating-add {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 100;
}

.products-grid {
  margin-top: 30px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 20px;
}

.product-card {
  background: white;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 5px 20px rgba(0,0,0,0.1);
}

.product-card img {
  width: 100%;
  height: 220px;
  object-fit: cover;
}

.product-info {
  padding: 15px;
}

.price {
  font-size: 22px;
  font-weight: bold;
  margin-top: 10px;
}

.category {
  color: gray;
}

.card-actions,
.edit-actions {
  display: flex;
  gap: 10px;
  padding: 15px;
}

.card-actions button,
.edit-actions button {
  flex: 1;
  border: none;
  padding: 12px;
  border-radius: 10px;
  cursor: pointer;
}

.plus {
  background: #111;
  color: white;
}

.minus {
  background: orange;
  color: white;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.4);
  backdrop-filter: blur(8px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
}

.modal {
  background: white;
  width: 500px;
  max-width: 100%;
  border-radius: 20px;
  padding: 25px;
}

.form-group {
  margin-top: 15px;
}

.form-group input,
.form-group select {
  width: 100%;
  margin-top: 8px;
  padding: 12px;
  border-radius: 10px;
  border: 1px solid #ccc;
}

.modal-actions {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.cart-panel,
.side-panel {
  position: fixed;
  top: 0;
  right: 0;
  width: 450px;
  max-width: 100%;
  height: 100vh;
  background: white;
  z-index: 1000;
  overflow-y: auto;
  padding: 20px;
  box-shadow: -5px 0 20px rgba(0,0,0,0.2);
}

.cart-header,
.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.cart-item {
  display: flex;
  gap: 10px;
  margin-top: 20px;
  background: #f7f7f7;
  padding: 10px;
  border-radius: 15px;
}

.cart-item img {
  width: 90px;
  height: 90px;
  object-fit: cover;
  border-radius: 10px;
}

.cart-actions-buttons {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.cart-actions-buttons button {
  border: none;
  padding: 8px;
  border-radius: 8px;
  cursor: pointer;
}

.cart-footer {
  margin-top: 20px;
}

.invoice-card,
.table-item {
  background: #f7f7f7;
  padding: 15px;
  border-radius: 15px;
  margin-top: 15px;
}

.table-form {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.table-form input,
.table-item input {
  flex: 1;
  padding: 12px;
  border-radius: 10px;
  border: 1px solid #ccc;
}

.notification {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: #111;
  color: white;
  padding: 15px 25px;
  border-radius: 12px;
  z-index: 3000;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.loading-screen {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  z-index: 5000;
  color: white;
}

.spinner {
  width: 70px;
  height: 70px;
  border: 6px solid rgba(255,255,255,0.2);
  border-top: 6px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}

@media (max-width: 500px) {
  .topbar {
    flex-direction: column;
    align-items: flex-start;
  }

  .search-box input {
    width: 100%;
  }

  .cart-item {
    flex-direction: column;
  }

  .cart-item img {
    width: 100%;
    height: 180px;
  }

  .modal {
    padding: 15px;
  }
}

@media (max-width: 300px) {
  .top-actions {
    flex-direction: column;
    width: 100%;
  }

  .top-actions button,
  .floating-add {
    width: 100%;
  }
}
</style>