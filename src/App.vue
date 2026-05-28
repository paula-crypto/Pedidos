<template>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
<header>
  <img src="./img/logo-fast-food-burger-cartoon-vector-removebg-preview.png" alt="logo">
  <div class="header-search">
    <input
      v-model="busqueda"
      type="text"
      placeholder="Buscar por nombre..."
      class="search-input"
      aria-label="Buscar por nombre"
    />
  </div>

  <div class="header-actions">
    <button type="button" class="btn-add" @click="toggleFormProducto" title="Agregar producto al menú"><i class="fas fa-plus"></i></button>
    <button @click="toggleModal()" class="carrito-btn">
      <i class="fas fa-shopping-cart"></i>
      <span v-if="carritoTotalItems > 0" class="carrito-badge" :class="{ mostrar: mostrarBadgeCarrito }">
        {{ carritoTotalItems }}
      </span>
    </button>
  </div>
</header>

<div class="categorias-bar">
  <button
    type="button"
    class="chip"
    :class="{ active: categoriaSeleccionada === 'Todas' }"
    @click="categoriaSeleccionada = 'Todas'"
  >
    Todas
  </button>
  <button
    v-for="cat in categorias"
    :key="cat"
    type="button"
    class="chip"
    :class="{ active: categoriaSeleccionada === cat }"
    @click="categoriaSeleccionada = cat"
  >
    {{ cat }}
  </button>
</div>


<div class="modal-backdrop" v-show="mostrarFormProducto" @click="toggleFormProducto" aria-hidden="true"></div>

<div class="agregar-producto-modal" v-show="mostrarFormProducto">
  <div class="modal-header">
    <h2>Agregar producto al menú</h2>
  <button type="button" class="modal-close" @click="toggleFormProducto" aria-label="Cerrar">
    <i class="fas fa-times" aria-hidden="true"></i>
  </button>
  </div>

  <form @submit.prevent="agregarProductoAlMenu">
    <input v-model="nuevoNombre" type="text" placeholder="Nombre del producto"  />
    <input v-model="nuevaImagen" type="url" placeholder="URL de la imagen"  />
    <input v-model="nuevaDescripcion" type="text" placeholder="Descripción"  />
    <label for="nuevoPrecio">Precio
    <input v-model.number="nuevoPrecio" type="number" min="0" step="0.01" placeholder="Precio"  />
    </label>
        <label for="nuevoPrecio">Cantidades
        
    <input v-model.number="nuevasUnidades" type="number" min="1" step="1" placeholder="Unidades"  />
    </label>
    <button type="submit">Agregar al menú</button>
  </form>
</div>

<div v-show="mostrarModal" class="ventana-modal" ref="modalCarrito">
    <div class="carrito-header">
      <h2>Carrito de Compras</h2>
      <button type="button" class="carrito-close" @click="toggleModal" aria-label="Cerrar">
        <i class="fas fa-times" aria-hidden="true"></i>
      </button>

    </div>
    <div v-if="mostrarExitoso" class="mensaje-exitoso">

      <i class="fas fa-check-circle"></i>
      <p>¡Compra realizada con éxito!</p>
    </div>
    <div v-else-if="procesando" class="spinner-container">
      <div class="spinner"></div>
      <p>Generando factura...</p>
    </div>
    <div v-else>
      <div v-if="carrito.length === 0" class="carrito-vacio">
        <p>El carrito está vacío</p>
      </div>
      <div v-else>
        <!-- Encabezados (una sola vez) -->
        <div class="carrito-table-header">
          <div class="item-info">
            <div class="col-producto"><span class="col-label">Producto</span></div>
            <div class="col-cantidad"><span class="col-label">Cantidad</span></div>
            <div class="col-precio"><span class="col-label">Dinero</span></div>
          </div>
        </div>

        <div v-for="(item, index) in carrito" :key="item.id" class="item-carrito">
          <div class="item-info">
            <div class="col-producto">
              <span>{{ item.nombre }}</span>
            </div>

            <div class="col-cantidad">
              <span class="qty-badge">{{ item.cantidad }}</span>
            </div>

            <div class="col-precio">
              {{ formatoMoneda(item.precio) }}

            </div>
          </div>

          <button class="btn-eliminar" @click="eliminarDelCarrito(index)" title="Eliminar producto">
            <i class="fas fa-trash-alt"></i>
          </button>
        </div>
        <div class="total">
<strong>Total: {{ formatoMoneda(total) }}</strong>

        </div>
        <div class="factura">
          <button @click="exportToPDF()">Comprar</button>
        </div>
      </div>
    </div>
</div>

<div id="menu">
    <div v-if="menuFiltrado.length === 0" class="carrito-vacio">
      <p>Producto no disponibles al filtrar</p>
    </div>

    <div v-for="plato in menuFiltrado" :class="['products', { agotado: plato.unidades === 0 }]">


      <img :src="plato.imagen" :alt="plato.nombre">
      <h1>{{ plato.nombre }}</h1>
      <p>{{ plato.descripcion }}</p>
<p class="precio">{{ formatoMoneda(plato.precio) }}</p>

      <button @click="agregarAlCarrito(plato)" :disabled="plato.unidades === 0">{{ plato.unidades === 0 ? 'Agotado' : 'Agregar al carrito' }}</button>
      <p class="unidad">Unidades: <span>{{ plato.unidades }}</span></p>
      <div v-if="plato.destacado" class="destacado-indicador">¡Nuevo!</div>

    </div>
</div>
</template>

<script setup>
import { computed, ref, watch } from 'vue';
import { jsPDF } from 'jspdf';
import Swal from 'sweetalert2';
import 'sweetalert2/dist/sweetalert2.min.css';
const busqueda = ref('');
const busquedaDebounced = ref('');
let debounceTimer = null;
const mostrarModal = ref(false);
const mostrarFormProducto = ref(false);
const procesando = ref(false);
const mostrarExitoso = ref(false);

const carrito = ref(cargarCarrito());

const carritoTotalItems = computed(() =>
  carrito.value.reduce((acc, item) => acc + item.cantidad, 0)
);

const mostrarBadgeCarrito = ref(false);

function syncBadgeVisibility() {
  mostrarBadgeCarrito.value = true;
}





const categoriaSeleccionada = ref('Todas');


function cargarCarrito() {
  const carritoGuardado = localStorage.getItem('carrito');
  return carritoGuardado ? JSON.parse(carritoGuardado) : [];
}

function cargarUnidades() {
  const unidadesGuardadas = localStorage.getItem('unidades');
  return unidadesGuardadas ? JSON.parse(unidadesGuardadas) : null;
}

const unidadesGuardadas = cargarUnidades();

const menu = ref([]);

const categorias = computed(() => {
  const set = new Set(menu.value.map(p => p.categoria).filter(Boolean));
  return Array.from(set);
});

const menuFiltrado = computed(() => {
  const q = busquedaDebounced.value;
  const cat = categoriaSeleccionada.value;

  return menu.value.filter(plato => {
    const coincideBusqueda = !q || plato.nombre.toLowerCase().includes(q);
    const coincideCategoria = cat === 'Todas' || plato.categoria === cat;
    return coincideBusqueda && coincideCategoria;
  });
});

watch(busqueda, (value) => {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    busquedaDebounced.value = value.trim().toLowerCase();
  }, 500);
  if (!value.trim()) {
    busquedaDebounced.value = '';
  }
});

let yaMostradoProductoNoDisponible = false;
watch(
  () => [busquedaDebounced.value, categoriaSeleccionada.value, menuFiltrado.value.length],
  ([q, _cat, len]) => {
    if (!q) {
      yaMostradoProductoNoDisponible = false;
      return;
    }

    if (len === 0 && !yaMostradoProductoNoDisponible) {
      yaMostradoProductoNoDisponible = true;
      Swal.fire({
        icon: 'warning',
        title: 'Producto no disponible',
        text: 'No se encontraron productos con la búsqueda actual. Verifica el nombre o prueba otro término.',
        confirmButtonText: 'Entendido',
        confirmButtonColor: '#22c55e',
        background: '#fff',
        iconColor: '#f59e0b',
      });
      return;
    }

    if (len > 0) {
      yaMostradoProductoNoDisponible = false;
    }
  }
);

function formatoMoneda(valor) {
  const num = typeof valor === 'number' ? valor : Number(valor);
  const formatter = new Intl.NumberFormat('es-CO', {
    style: 'currency',
    currency: 'COP',
    currencyDisplay: 'symbol',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
    useGrouping: true,
  });

  return Number.isNaN(num) ? formatter.format(0) : formatter.format(num);
}




menu.value = [
  { id: 1, nombre: "Hamburguesa", categoria: 'Hamburguesas', imagen: "https://png.pngtree.com/png-vector/20250429/ourmid/pngtree-burger-image-with-white-background-png-image_16049638.png", descripcion: "Carne de res, cebolla salteada, lechuga, tomate y papas", precio: 12.50, unidades: unidadesGuardadas?.[0] ?? 15 },
  { id: 2, nombre: "Hamburguesa Doble", categoria: 'Hamburguesas', imagen: "https://png.pngtree.com/png-clipart/20240321/original/pngtree-double-cheese-burger-png-image_14644513.png", descripcion: "Doble carne, doble queso y vegetales", precio: 15.00, unidades: unidadesGuardadas?.[1] ?? 15 },
  { id: 3, nombre: "Hamburguesa Triple", categoria: 'Hamburguesas', imagen: "https://png.pngtree.com/png-clipart/20250507/original/pngtree-triple-cheeseburger-delicious-stacked-food-isolated-png-image_20937321.png", descripcion: "Triple carne,queso y vegetales", precio: 8.00, unidades: unidadesGuardadas?.[2] ?? 15 },
  { id: 4, nombre: "Perro Caliente", categoria: 'Hot Dogs', imagen: "https://png.pngtree.com/png-clipart/20241129/original/pngtree-hot-dog-with-mustard-and-ketchup-isolated-on-a-transparent-background-png-image_17419880.png", descripcion: "Perro caliente con salchicha, tocino y salsas", precio: 10.00, unidades: unidadesGuardadas?.[3] ?? 15 },

  { id: 5, nombre: "Salchipapa", categoria: 'Papas', imagen: "https://static.vecteezy.com/system/resources/thumbnails/032/325/506/small/french-fries-with-cheese-and-bacon-isolated-on-transparent-background-file-cut-out-ai-generated-png.png", descripcion: "Papas crujientes con salchicha, queso y salsas", precio: 9.50, unidades: unidadesGuardadas?.[4] ?? 15 },

  { id: 6, nombre: "Patacón relleno", categoria: 'Patacones', imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQdGzdMHi52Cq74JCCzjrXvRg9PY3aCyugPHA&s", descripcion: "Plátano frito relleno de carne y queso", precio: 11.00, unidades: unidadesGuardadas?.[5] ?? 15 },
  { id: 7, nombre: "Tacos de Pollo", categoria: 'Tacos', imagen: "https://png.pngtree.com/png-vector/20241110/ourmid/pngtree-authentic-chicken-tacos-with-cilantro-png-image_14338727.png", descripcion: "Tacos con pollo desmenuzado y vegetales frescos", precio: 9.00, unidades: unidadesGuardadas?.[6] ?? 15 },
  { id: 8, nombre: "Quesadilla", categoria: 'Tacos', imagen: "https://png.pngtree.com/png-clipart/20241003/original/pngtree-hearty-mexican-quesadilla-perfectly-grilled-with-meat-and-veggies-png-image_16190692.png", descripcion: "Tortilla con queso fundido y relleno a elegir", precio: 8.50, unidades: unidadesGuardadas?.[7] ?? 15 },

  { id: 9, nombre: "Pizza Personal", categoria: 'Pizzas', imagen: "https://png.pngtree.com/png-clipart/20250106/original/pngtree-yummy-stretchy-cheese-pepperoni-pizza-png-image_19960043.png", descripcion: "Pizza pequeña con queso y peperoni", precio: 13.00, unidades: unidadesGuardadas?.[8] ?? 15 },
  { id: 10, nombre: "Sandwich Cubano", categoria: 'Sandwiches', imagen: "https://static.vecteezy.com/system/resources/thumbnails/055/669/484/small/delicious-cuban-sandwich-with-layers-of-meats-and-melted-cheese-served-fresh-png.png", descripcion: "Pan con jamón, cerdo y queso derretido", precio: 10.50, unidades: unidadesGuardadas?.[9] ?? 15 },
  { id: 11, nombre: "Pollo Frito", categoria: 'Pollo', imagen: "https://png.pngtree.com/png-clipart/20231017/original/pngtree-fried-chicken-american-food-png-image_13325963.png", descripcion: "Pollo crujiente con papas y ensalada", precio: 11.50, unidades: unidadesGuardadas?.[10] ?? 15 },
  { id: 12, nombre: "Nuggets de Pollo", categoria: 'Pollo', imagen: "https://png.pngtree.com/png-clipart/20250104/original/pngtree-mouth-watering-fried-chicken-nuggets-and-fries-for-fast-food-lovers-png-image_18953065.png", descripcion: "Nuggets dorados con salsa y papas", precio: 7.50, unidades: unidadesGuardadas?.[11] ?? 15 },

  { id: 13, nombre: "Empanadas", categoria: 'Empanadas', imagen: "https://toppng.com/uploads/preview/empanadas-colombianas-png-empanada-de-maiz-11562940498m2ioxfluel.png", descripcion: "Empanadas de carne o queso, 3 unidades", precio: 6.00, unidades: unidadesGuardadas?.[12] ?? 15 },
  { id: 14, nombre: "Alitas BBQ", categoria: 'Pollo', imagen: "https://png.pngtree.com/png-clipart/20250111/original/pngtree-realistic-high-quality-bbq-chicken-wings-platter-png-image_19087161.png", descripcion: "Alitas bañadas en salsa BBQ picante", precio: 10.00, unidades: unidadesGuardadas?.[13] ?? 15 },
  { id: 15, nombre: "Nachos con Queso", categoria: 'Tacos', imagen: "https://toppng.com/uploads/preview/ork-sausage-nachos-nachos-con-carne-y-queso-115635741348pfqgjldlr.png", descripcion: "Nachos crujientes con queso derretido y jalapeños", precio: 8.00, unidades: unidadesGuardadas?.[14] ?? 15 },
  { id: 16, nombre: "Camarones Fritos", categoria: 'Mariscos', imagen: "https://static.vecteezy.com/system/resources/thumbnails/050/738/203/small/high-quality-butterfly-shrimps-or-fried-prawns-that-look-delicious-isolated-on-transparent-background-for-menus-png.png", descripcion: "Camarones Fritos con salsa de la casa", precio: 14.00, unidades: unidadesGuardadas?.[15] ?? 15 },

  { id: 17, nombre: "Ceviche", categoria: 'Mariscos', imagen: "https://png.pngtree.com/png-clipart/20250122/original/pngtree-healthy-shrimp-ceviche-with-fresh-herbs-and-veggies-png-image_19983518.png", descripcion: "Pescado fresco marinado en limón y especias", precio: 12.00, unidades: unidadesGuardadas?.[16] ?? 15 },
  { id: 18, nombre: "Arepa Rellena", categoria: 'Arepas', imagen: "https://png.pngtree.com/png-clipart/20190630/original/pngtree-stuffed-arepa-png-image_4172135.jpg", descripcion: "Arepa con queso, Carne y salchicha", precio: 7.00, unidades: unidadesGuardadas?.[17] ?? 15 },
  { id: 19, nombre: "Churros", categoria: 'Postres', imagen: "https://img.freepik.com/psd-gratis/delicious-churros-with-rich-melted-chocolate-sauce_84443-72596.jpg?semt=ais_hybrid&w=740&q=80", descripcion: "Churros crujientes con chocolate caliente", precio: 5.50, unidades: unidadesGuardadas?.[18] ?? 15 },
  { id: 20, nombre: "Pastel de Tres Leches", categoria: 'Postres', imagen: "https://img.freepik.com/psd-gratis/delicioso-pastel-crema-coco-rebanada-dulce-regalo_191095-86310.jpg?semt=ais_hybrid&w=740&q=80", descripcion: "Pastel húmedo de tres leches", precio: 6.50, unidades: unidadesGuardadas?.[19] ?? 15 },

  { id: 21, nombre: "Flan de Caramelo", categoria: 'Postres', imagen: "https://png.pngtree.com/png-clipart/20250123/original/pngtree-delicious-spanish-flan-with-rich-caramel-sauce-on-a-plate-png-image_19996289.png", descripcion: "Flan casero con caramelo tradicional", precio: 5.00, unidades: unidadesGuardadas?.[20] ?? 15 },
  { id: 22, nombre: "Batido de Frutas", categoria: 'Bebidas', imagen: "https://img.freepik.com/vector-premium/batido-coctel-jugo-fresa-o-yogur-vaso-paja-entero-mitad-fresa-aislado-sobre-fondo-transparente-ilustracion-vector-3d-realista_545793-1246.jpg", descripcion: "Batidos frescos de frutas de temporada", precio: 4.50, unidades: unidadesGuardadas?.[21] ?? 15 },
  { id: 23, nombre: "Jugo Natural", categoria: 'Bebidas', imagen: "https://png.pngtree.com/png-clipart/20250609/original/pngtree-refreshing-fruit-smoothies-with-strawberries-mango-and-orange-png-image_21148205.png", descripcion: "Jugo natural recién exprimido", precio: 3.50, unidades: unidadesGuardadas?.[22] ?? 15 },
  { id: 24, nombre: "Agua Fresca", categoria: 'Bebidas', imagen: "https://png.pngtree.com/png-vector/20231116/ourmid/pngtree-glass-of-water-natural-png-image_10480871.png", descripcion: "Agua fresca ", precio: 2.00, unidades: unidadesGuardadas?.[23] ?? 15 },

  { id: 25, nombre: "Ensalada César", categoria: 'Ensaladas', imagen: "https://png.pngtree.com/png-clipart/20250102/original/pngtree-caesar-salad-dish-png-image_18599836.png", descripcion: "Ensalada fresca con pechuga de pollo", precio: 9.00, unidades: unidadesGuardadas?.[24] ?? 15 },
  { id: 26, nombre: "Sushi Roll", categoria: 'Mariscos', imagen: "https://static.vecteezy.com/system/resources/thumbnails/054/065/072/small_2x/high-resolution-sushi-roll-illustration-transparent-background-for-food-themed-art-png.png", descripcion: "Roll de sushi fresco con pescado y vegetales", precio: 11.00, unidades: unidadesGuardadas?.[25] ?? 15 },
  { id: 27, nombre: "Fideos a la Carbonara", categoria: 'Pastas', imagen: "https://static.vecteezy.com/system/resources/previews/056/615/020/non_2x/spaghetti-carbonara-top-view-isolate-on-transparent-background-png.png", descripcion: "Fideos en salsa cremosa con tocino", precio: 10.50, unidades: unidadesGuardadas?.[26] ?? 15 },
  { id: 28, nombre: "Poutine", categoria: 'Papas', imagen: "https://static.vecteezy.com/system/resources/previews/055/299/651/non_2x/delicious-poutine-plate-on-transparent-background-png.png", descripcion: "Papas fritas con queso fundido y salsa", precio: 8.50, unidades: unidadesGuardadas?.[27] ?? 15 },


  { id: 29, nombre: "Tarta de Queso", categoria: 'Postres', imagen: "https://img.freepik.com/fotos-premium/rebanada-queso-corteza-blanca_1019429-43225.jpg?semt=ais_hybrid&w=740&q=80", descripcion: "Tarta de queso cremosa ", precio: 7.00, unidades: unidadesGuardadas?.[28] ?? 15 },
  { id: 30, nombre: "Helado Artesanal", categoria: 'Postres', imagen: "https://png.pngtree.com/png-clipart/20250108/original/pngtree-flavorful-ice-cream-collection-with-fresh-berries-png-image_19755817.png", descripcion: "Helado artesanal en varios sabores", precio: 5.50, unidades: unidadesGuardadas?.[29] ?? 15 }
];

const nuevoNombre = ref('');
const nuevaImagen = ref('');
const nuevaDescripcion = ref('');
const nuevoPrecio = ref(0);
const nuevasUnidades = ref(1);

function toggleModal() {
  mostrarModal.value = !mostrarModal.value;
  syncBadgeVisibility();
}

function agregarProductoAlMenu() {
  const nombre = nuevoNombre.value.trim();
  const imagen = nuevaImagen.value.trim();
  const descripcion = nuevaDescripcion.value.trim();
  const precio = parseFloat(nuevoPrecio.value);
  const unidades = parseInt(nuevasUnidades.value, 10);

  // Categoría por nombre (rápido para mantener el filtro)
  let categoria = '';
  const n = nombre.toLowerCase();
  if (n.includes('hamburguesa')) categoria = 'Hamburguesas';
  else if (n.includes('salchipapa') || n.includes('poutine') || n.includes('papas')) categoria = 'Papas';
  else if (n.includes('perro')) categoria = 'Hot Dogs';

  else if (n.includes('patacón') || n.includes('patacon')) categoria = 'Patacones';
  else if (n.includes('tacos') || n.includes('quesadilla') || n.includes('nachos')) categoria = 'Tacos';
  else if (n.includes('pizza')) categoria = 'Pizzas';
  else if (n.includes('sandwich')) categoria = 'Sandwiches';
  else if (n.includes('pollo') || n.includes('nuggets') || n.includes('alitas')) categoria = 'Pollo';
  else if (n.includes('empanadas')) categoria = 'Empanadas';
  else if (n.includes('ceviche') || n.includes('sushi') || n.includes('camarones') || n.includes('mar')) categoria = 'Mariscos';
  else if (n.includes('arepa')) categoria = 'Arepas';
  else if (n.includes('fideos') || n.includes('carbonara') || n.includes('poutine')) categoria = 'Pastas';
  else if (n.includes('ensalada')) categoria = 'Ensaladas';
  else if (n.includes('churros') || n.includes('flan') || n.includes('tres leches') || n.includes('helado') || n.includes('pastel') || n.includes('tarta')) categoria = 'Postres';
  else if (n.includes('jugo') || n.includes('batido') || n.includes('agua fresca')) categoria = 'Bebidas';


  if (!nombre || !imagen || !descripcion || isNaN(precio) || precio <= 0 || isNaN(unidades) || unidades <= 0) {
    Swal.fire({
      icon: 'error',
      title: 'Datos incompletos',
      text: 'Ingresa todos los datos del producto correctamente.',
      confirmButtonText: 'Entendido'
    });
    return;
  }

  const nuevoId = menu.value.length > 0 ? Math.max(...menu.value.map(item => item.id)) + 1 : 1;

  menu.value.push({
    id: nuevoId,
    nombre,
    categoria,
    imagen,
    descripcion,
    precio,
    unidades,
    destacado: true
  });

  // Quitamos el resaltado después de un tiempo
  setTimeout(() => {
    const item = menu.value.find(p => p.id === nuevoId);
    if (item) item.destacado = false;
  }, 2000);



  // Mensaje de éxito al agregar el producto al menú
  Swal.fire({
    position: 'center',
    icon: 'success',
    title: 'Producto agregado',
    html: `<div style="font-size:16px">✅ <b>${nombre}</b> se agregó exitosamente al menú.</div>`,
    confirmButtonText: 'OK',
    showConfirmButton: true,
    allowOutsideClick: false,
    backdrop: true
  });

  nuevoNombre.value = '';
  nuevaImagen.value = '';
  nuevaDescripcion.value = '';
  nuevoPrecio.value = 0;
  nuevasUnidades.value = 1;
  mostrarFormProducto.value = false;
}

function toggleFormProducto() {
  mostrarFormProducto.value = !mostrarFormProducto.value;
}

function guardarCarrito() {
  localStorage.setItem('carrito', JSON.stringify(carrito.value));
  syncBadgeVisibility();
}

function agregarAlCarrito(plato) {
  if (plato.unidades == 0) {
    Swal.fire({
      icon: 'warning',
      title: 'Agotado',
      text: 'No hay unidades disponibles',
      confirmButtonText: 'Entendido'
    });
    return;
  }

  const itemExistente = carrito.value.find(item => item.id === plato.id);

  if (itemExistente) {
    itemExistente.cantidad++;
  } else {
    carrito.value.push({
      id: plato.id,
      nombre: plato.nombre,
      precio: plato.precio,
      cantidad: 1
    });
  }

  plato.unidades--;
  guardarCarrito();
  guardarUnidades();

  // (Solo badge encima del botón del carrito)
  syncBadgeVisibility();


}

function guardarUnidades() {
  const unidades = menu.value.map(plato => plato.unidades);
  localStorage.setItem('unidades', JSON.stringify(unidades));
}

const total = computed(() =>
  carrito.value.reduce((acc, item) => acc + (item.precio * item.cantidad), 0)
);


function eliminarDelCarrito(index) {
  const itemEliminado = carrito.value[index];
  
  // Disminuir cantidad en 1
  if (itemEliminado.cantidad > 1) {
    itemEliminado.cantidad--;
  } else {
    // Si la cantidad es 1, eliminar el item
    carrito.value.splice(index, 1);
  }
  
  // Encontrar el producto en el menú y restaurar 1 unidad
  const productoEnMenu = menu.value.find(plato => plato.id === itemEliminado.id);
  if (productoEnMenu) {
    productoEnMenu.unidades += 1;
  }
  
  // Guardar cambios
  guardarCarrito();
  guardarUnidades();
}
function exportToPDF() {
  procesando.value = true;

  const logoUrl = new URL('./img/logo-fast-food-burger-cartoon-vector-removebg-preview.png', import.meta.url).href;
  const logo = new Image();
  logo.src = logoUrl;
  logo.onload = () => generarPDF(logo);
  logo.onerror = () => generarPDF();

  function generarPDF(logoImage) {
    const doc = new jsPDF({
      orientation: 'portrait',
      unit: 'mm',
      format: 'letter'
    });
    doc.setDocumentProperties({
      title: 'Factura de Venta - Fast Food Burger',
      author: 'Fast Food Burger',
      subject: 'Factura de Venta',
      keywords: 'factura, venta, restaurante'
    });

    const pageWidth = doc.internal.pageSize.getWidth();
    const pageHeight = doc.internal.pageSize.getHeight();
    const margin = 20;
    const rightX = pageWidth - margin;
    const contentWidth = rightX - margin;
    let currentPage = 1;

    const primaryColor = [20, 60, 120];
    const darkText = [40, 40, 40];
    const mediumGray = [120, 120, 120];
    const lightGray = [240, 240, 240];
    const borderGray = [200, 200, 200];

    const fecha = new Date();
    const facturaNum = `FAC-${fecha.getFullYear()}${String(fecha.getMonth() + 1).padStart(2, '0')}${String(fecha.getDate()).padStart(2, '0')}-${String(Math.floor(Math.random() * 10000)).padStart(4, '0')}`;

    function drawHeader() {
      let headerY = 15;

      // Logo area
      let logoX = margin;
      if (logoImage) {
        doc.addImage(logoImage, 'PNG', margin, headerY, 24, 24);
        logoX = margin + 30;
      }

      // Restaurant name and info (left side)
      doc.setFont('helvetica', 'bold');
      doc.setFontSize(18);
      doc.setTextColor(...primaryColor);
      doc.text('FAST FOOD BURGER', logoX, headerY + 6);

      doc.setFont('helvetica', 'normal');
      doc.setFontSize(10);
      doc.setTextColor(...mediumGray);
      doc.text('Restaurante de Comida Rápida', logoX, headerY + 14);
      doc.text('NIT: 900.123.456-7', logoX, headerY + 20);
      doc.text('Calle 123 # 45-67, Ciudad', logoX, headerY + 26);
      doc.text('Tel: (123) 456-7890', logoX, headerY + 32);


      // Factura info box (right side)
      const boxW = 75;
      const boxH = 42;
      const boxX = rightX - boxW;
      const boxY = headerY - 2;

      doc.setFillColor(255, 255, 255);
      doc.setDrawColor(...primaryColor);
      doc.setLineWidth(0.6);
      doc.rect(boxX, boxY, boxW, boxH);

      let by = boxY + 8;
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(13);
    doc.setTextColor(...primaryColor);
    doc.text('FACTURA DE VENTA', boxX + boxW / 2, by, { align: 'center' });

    by += 7;
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.3);
    doc.line(boxX + 5, by, boxX + boxW - 5, by);

    by += 7;
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(10);
    doc.setTextColor(...darkText);
    doc.text(`No.  ${facturaNum}`, boxX + 6, by);
    by += 6;
    doc.text(`Fecha:  ${fecha.toLocaleDateString('es-ES')}`, boxX + 6, by);
    by += 6;
    doc.text(`Hora:  ${fecha.toLocaleTimeString('es-ES')}`, boxX + 6, by);


      // Horizontal separator line
      return headerY + 42;
    }

    function drawFooter() {
      const footerY = pageHeight - 22;

      doc.setDrawColor(...borderGray);
      doc.setLineWidth(0.4);
      doc.line(margin, footerY, rightX, footerY);

      doc.setFont('helvetica', 'italic');
      doc.setFontSize(10);
      doc.setTextColor(...mediumGray);
      doc.text('Gracias por su compra. Vuelva pronto.', margin, footerY + 5);
      doc.text('Este documento es un comprobante de pago válido.', margin, footerY + 11);
      doc.text(`Página ${currentPage}`, rightX, footerY + 5, { align: 'right' });

    }

    let y = drawHeader();

    // Separator line after header
    y += 4;
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.5);
    doc.line(margin, y, rightX, y);

    // CLIENTE section
    y += 12;
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(12);
    doc.setTextColor(...primaryColor);
    doc.text('DATOS DEL CLIENTE', margin, y);

    y += 3;
    doc.setDrawColor(...primaryColor);
    doc.setLineWidth(0.4);
    doc.line(margin, y, margin + 65, y);

    y += 8;
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(11);
    doc.setTextColor(...darkText);
    doc.text('Nombre:     Consumidor Final', margin, y);
    y += 7;
    doc.text('Identificación:     222222222222', margin, y);
    y += 7;
    doc.text('Dirección:     Ciudad', margin, y);
    y += 7;
    doc.text('Teléfono:     N/A', margin, y);


    // Separator before table
    y += 10;
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.5);
    doc.line(margin, y, rightX, y);

    // TABLE
    y += 12;
    const colDescW = contentWidth * 0.48;
    const colCantW = contentWidth * 0.14;
    const colUnitW = contentWidth * 0.19;
    const colSubW = contentWidth * 0.19;

    const col1X = margin;
    const col2X = col1X + colDescW;
    const col3X = col2X + colCantW;
    const col4X = col3X + colUnitW;

    // Table header background
    doc.setFillColor(...lightGray);
    doc.rect(margin, y - 5, contentWidth, 8, 'F');

    // Table header borders
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.3);
    doc.rect(margin, y - 5, contentWidth, 8);
    doc.line(col2X, y - 5, col2X, y + 3);
    doc.line(col3X, y - 5, col3X, y + 3);
    doc.line(col4X, y - 5, col4X, y + 3);

    doc.setFont('helvetica', 'bold');
    doc.setFontSize(10);
    doc.setTextColor(...darkText);
    doc.text('DESCRIPCIÓN', col1X + 3, y + 1);
    doc.text('CANT.', col2X + colCantW / 2, y + 1, { align: 'center' });
    doc.text('P. UNIT.', col3X + colUnitW / 2, y + 1, { align: 'center' });
    doc.text('SUBTOTAL', col4X + colSubW / 2, y + 1, { align: 'center' });

    y += 9;
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(11);


    carrito.value.forEach((item, index) => {
      const rowH = 9;


      // Page break check
      if (y + rowH > pageHeight - margin - 50) {
        drawFooter();
        doc.addPage();
        currentPage++;
        y = drawHeader();
        y += 4;
        doc.setDrawColor(...borderGray);
        doc.setLineWidth(0.5);
        doc.line(margin, y, rightX, y);
        y += 12;
      }

      // Alternating row background
      if (index % 2 === 1) {
        doc.setFillColor(250, 250, 250);
        doc.rect(margin, y - 4, contentWidth, rowH, 'F');
      }

      const subtotal = item.precio * item.cantidad;
      const maxChars = Math.floor(colDescW / 2.1);
      const nombre = item.nombre.length > maxChars ? item.nombre.substring(0, maxChars - 3) + '...' : item.nombre;

      doc.setTextColor(...darkText);
      doc.text(nombre, col1X + 3, y + 1);
      doc.text(item.cantidad.toString(), col2X + colCantW / 2, y + 1, { align: 'center' });
      doc.text(new Intl.NumberFormat('es-CO', {
        style: 'currency',
        currency: 'COP',
        currencyDisplay: 'symbol',
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
        useGrouping: true,
      }).format(item.precio), col3X + colUnitW / 2, y + 1, { align: 'center' });
      doc.text(new Intl.NumberFormat('es-CO', {
        style: 'currency',
        currency: 'COP',
        currencyDisplay: 'symbol',
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
        useGrouping: true,
      }).format(subtotal), col4X + colSubW / 2, y + 1, { align: 'center' });


      // Row bottom border
      doc.setDrawColor(...borderGray);
      doc.setLineWidth(0.2);
      doc.line(margin, y + 4, rightX, y + 4);
      doc.line(col2X, y - 4, col2X, y + 4);
      doc.line(col3X, y - 4, col3X, y + 4);
      doc.line(col4X, y - 4, col4X, y + 4);

      y += rowH;
    });

    // Separator before totals
    y += 6;
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.5);
    doc.line(margin, y, rightX, y);

    // TOTALS section (right aligned)
    y += 10;
    const totalValue = total.value;

    const totW = 70;
    const totX = rightX - totW;

    // Subtotal
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(11);
    doc.setTextColor(...darkText);
    doc.text('Subtotal:', totX + 5, y);
    doc.text(new Intl.NumberFormat('es-CO', {
      style: 'currency',
      currency: 'COP',
      currencyDisplay: 'symbol',
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
      useGrouping: true,
    }).format(totalValue), rightX - 5, y, { align: 'right' });




    y += 7;
    // Impuestos
    doc.text('Impuestos (0%):', totX + 5, y);
    doc.text(new Intl.NumberFormat('es-CO', {
      style: 'currency',
      currency: 'COP',
      currencyDisplay: 'symbol',
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
      useGrouping: true,
    }).format(0), rightX - 5, y, { align: 'right' });


    y += 5;
    // Line before total
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.3);
    doc.line(totX + 5, y, rightX - 5, y);

    y += 8;
    // Total a pagar (highlighted)
    doc.setFillColor(...lightGray);
    doc.rect(totX - 2, y - 5, totW + 2, 10, 'F');
    doc.setDrawColor(...primaryColor);
    doc.setLineWidth(0.5);
    doc.rect(totX - 2, y - 5, totW + 2, 10);

    doc.setFont('helvetica', 'bold');
    doc.setFontSize(12);
    doc.setTextColor(...primaryColor);
    doc.text('TOTAL A PAGAR', totX + 5, y + 1);
    doc.text(new Intl.NumberFormat('es-CO', {
      style: 'currency',
      currency: 'COP',
      currencyDisplay: 'symbol',
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
      useGrouping: true,
    }).format(totalValue), rightX - 5, y + 1, { align: 'right' });



    // Amount in words (conventional)
    y += 14;
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(10);
    doc.setTextColor(...mediumGray);
    doc.text(`Valor en letras:  ${numeroALetras(totalValue)} PESOS M/C.`, margin, y);


    // Observation section

    y += 10;
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(11);
    doc.setTextColor(...primaryColor);
    doc.text('OBSERVACIONES', margin, y);

    y += 4;
    doc.setDrawColor(...borderGray);
    doc.setLineWidth(0.3);
    doc.line(margin, y, margin + 45, y);

    y += 6;
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(10);
    doc.setTextColor(...mediumGray);
    doc.text('Pago realizado en efectivo. No se aceptan devoluciones.', margin, y);


    // Authorized signature area
    y += 20;
    const sigX = margin + contentWidth * 0.6;
    doc.setDrawColor(...darkText);
    doc.setLineWidth(0.3);
    doc.line(sigX, y, rightX, y);
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(8);
    doc.setTextColor(...mediumGray);
    doc.text('Firma Autorizada', sigX + 20, y + 5);

    // Footer
    drawFooter();

    doc.output('dataurlnewwindow');
    finalizarPDF();
  }


  function numeroALetras(numero) {
    const unidades = ['', 'UNO', 'DOS', 'TRES', 'CUATRO', 'CINCO', 'SEIS', 'SIETE', 'OCHO', 'NUEVE'];
    const decenas = ['', 'DIEZ', 'VEINTE', 'TREINTA', 'CUARENTA', 'CINCUENTA', 'SESENTA', 'SETENTA', 'OCHENTA', 'NOVENTA'];
    const especiales = ['DIEZ', 'ONCE', 'DOCE', 'TRECE', 'CATORCE', 'QUINCE', 'DIECISÉIS', 'DIECISIETE', 'DIECIOCHO', 'DIECINUEVE'];

    const entero = Math.floor(numero);
    const decimal = Math.round((numero - entero) * 100);

    if (entero === 0) return 'CERO';
    if (entero < 10) return unidades[entero];
    if (entero < 20) return especiales[entero - 10];
    if (entero < 100) {
      const d = Math.floor(entero / 10);
      const u = entero % 10;
      return u === 0 ? decenas[d] : `${decenas[d]} Y ${unidades[u]}`;
    }
    if (entero < 1000) {
      const c = Math.floor(entero / 100);
      const resto = entero % 100;
      const centena = c === 1 ? 'CIEN' : `${unidades[c]}CIENTOS`;
      return resto === 0 ? centena : `${centena} ${numeroALetras(resto)}`;
    }
    if (entero < 1000000) {
      const m = Math.floor(entero / 1000);
      const resto = entero % 1000;
      const miles = m === 1 ? 'MIL' : `${numeroALetras(m)} MIL`;
      return resto === 0 ? miles : `${miles} ${numeroALetras(resto)}`;
    }
    return 'UN MILLÓN';
  }

  function finalizarPDF() {
    setTimeout(() => {
      procesando.value = false;
      mostrarExitoso.value = true;
      carrito.value = [];
      localStorage.removeItem('carrito');
      setTimeout(() => {
        mostrarExitoso.value = false;
        mostrarModal.value = false;
      }, 2000);
    }, 500);
  }
}

</script>

<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body, html {
    background-size: cover;
    min-height: 100vh;
  }

  header {
    background-color: #000000;
    padding: 20px 40px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
    border-bottom: 3px solid #ffd700;
    flex-wrap: wrap;
    gap: 12px;
  }

  header img {
    height: 98px;
    width: auto;
    max-width: 100%;
  }

  .header-search {
    flex: 1;
    display: flex;
    justify-content: center;
    padding: 0 20px;
  }

  .search-input {
    width: min(420px, 100%);
    padding: 10px 14px;
    border-radius: 10px;
    border: 1px solid #333;
    outline: none;
    font-size: 1em;
  }

  .search-input:focus {
    border-color: #ffd700;
    box-shadow: 0 0 0 3px rgba(255, 215, 0, 0.25);
  }

  .header-search {
    flex: 1;
    display: flex;
    justify-content: center;
    padding: 0 20px;
  }

  .search-input {
    width: min(420px, 100%);
    padding: 10px 14px;
    border-radius: 10px;
    border: 1px solid #333;
    outline: none;
    font-size: 1em;
  }

  .search-input:focus {
    border-color: #ffd700;
    box-shadow: 0 0 0 3px rgba(255, 215, 0, 0.25);
  }

  #menu {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
  padding: 40px 20px;
  background: rgba(0, 0, 0, 0.55);
  min-height: calc(100vh - 100px);
}

.products {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  text-align: center;
  background-color: #f9f9f9;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  cursor: pointer;
  width: 250px;
  flex-shrink: 0;
}

.products:hover {
  transform: scale(1.05);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  background-color: #fff;
}

.products.agotado {
  opacity: 0.45;
  filter: grayscale(80%);
  cursor: not-allowed;
}

.products.agotado button {
  background-color: #999;
  cursor: not-allowed;
}

.products img {
  max-width: 100%;
  height: auto;
  border-radius: 4px;
}

.products h1 {
  font-size: 1.2em;
  margin: 10px 0;
}

.products p {
  font-size: 0.9em;
  color: #666;
}

.precio {
  font-size: 1.3em;
  color: #27ae60;
  font-weight: bold;
  margin: 10px 0;
}

header .header-actions {
  display: flex;
  gap: 12px;
  align-items: center;
}

header .btn-add,
header button {
  background-color: #22c55e;
  padding: 10px 16px;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

header .btn-add:hover,
header button:hover {
  background-color: #0ea5e9;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.45);
  z-index: 900;
}

.agregar-producto-modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: min(520px, calc(100% - 40px));
  background: white;
  border-radius: 16px;
  box-shadow: 0 18px 45px rgba(0, 0, 0, 0.25);
  padding: 24px;
  z-index: 950;
}

.agregar-producto-modal .modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
}

.agregar-producto-modal .modal-close {
  background: transparent;
  border: none;
  color: #333;
  font-size: 18px;
  cursor: pointer;
}

.agregar-producto-modal form {
  display: grid;
  gap: 12px;
}

.agregar-producto-modal input {
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 10px;
  padding: 12px 14px;
  font-size: 0.95em;
}

.agregar-producto-modal button[type="submit"] {
  width: 100%;
  background-color: #22c55e;
  color: white;
  border: none;
  padding: 14px;
  border-radius: 10px;
  cursor: pointer;
  font-weight: 700;
}

.agregar-producto-modal button[type="submit"]:hover {
  background-color: #0ea5e9;
}

.ventana-modal {
  position: fixed;
  top: 1%;

  left: 50%;
  transform: translateX(-50%);

  background-color: white;
  border: 2px solid #333;
  border-radius: 8px;
  padding: 20px;
  min-width: 400px;
  max-height: 400px;
  overflow-y: auto;
  z-index: 1000;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.ventana-modal h2 {
  margin-top: 0;
  color: #333;
}




.carrito-vacio {
  text-align: center;
  padding: 20px;
  color: #999;
}

  .item-carrito {
  display: flex;
  justify-content: space-between;
  padding: 10px 0;
  border-bottom: 1px solid #eee;
}

  .item-info {
    display: grid;
    grid-template-columns: 1.2fr 0.7fr 0.8fr;
    align-items: center;
    gap: 10px;
  }

  .col-precio {
    display: flex;
    flex-direction: column;
    gap: 2px;
    align-items: flex-start;
  }

  .carrito-table-header .col-precio {
    transform: none;
  }







  .col-producto,
  .col-cantidad,
  .col-precio {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

.col-label {
    font-size: 11px;
    font-weight: 900;
    color: #000;
    opacity: 0.95;
  }



.qty-badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 28px;
    padding: 2px 8px;
    border-radius: 999px;
    background: rgba(255, 215, 0, 0.18);
    border: 1px solid rgba(255, 215, 0, 0.55);
    color: #000;
    font-weight: 900;
  }


.unit-price {
    font-weight: 800;
    color: #0b3d2e;
    display: block;
    width: 100%;
    text-align: right;
    padding-right: 4px;

    /* Ajuste visual: mover un poquito hacia la izquierda y mantener el valor bajo el header */
    position: relative;
    left: -24px;
  }






.carrito-table-header {
    padding: 10px 0 0;
    border-bottom: 1px solid #eee;
    margin-bottom: 6px;
  }

.carrito-table-header .col-label {
    font-size: 16px;
  }




  .carrito-btn {
    position: relative;
  }

  .carrito-badge {
    position: absolute;
    top: -8px;
    right: -8px;
    background: #ffd700;
    color: #000;
    font-weight: 900;
    font-size: 12px;
    line-height: 1;
    border-radius: 999px;
    padding: 6px 8px;
    opacity: 0;
    transform: scale(0.9);
    transition: opacity 0.2s ease, transform 0.2s ease;
    pointer-events: none;
  }

  .carrito-badge.mostrar {
    opacity: 1;
    transform: scale(1);
  }



.total {
  margin-top: 15px;
  padding-top: 10px;
  border-top: 2px solid #ffd700;
  text-align: right;
  font-size: 1.2em;
  color: #0b3d2e;
  font-weight: 800;
  background: rgba(255, 215, 0, 0.12);
  padding-left: 10px;
  border-radius: 10px;
}


.spinner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  gap: 20px;
}

.spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #27ae60;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.spinner-container p {
  color: #333;
  font-size: 1.1em;
}

.mensaje-exitoso {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  gap: 15px;
}

.mensaje-exitoso i {
  font-size: 3em;
  color: #27ae60;
}

.mensaje-exitoso p {
  font-size: 1.3em;
  color: #27ae60;
  font-weight: bold;
}


.categorias-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: center;
  padding: 16px 20px;
  background: rgba(0, 0, 0, 0.55);
  border-bottom: 1px solid rgba(255, 215, 0, 0.25);
}

.chip {
  border: 1px solid rgba(255, 215, 0, 0.5);
  background: rgba(0, 0, 0, 0.25);
  color: #fff;
  padding: 8px 14px;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 600;
}

 .chip.active {
  background: #ffd700;
  color: #000;
  border-color: #ffd700;
}

.destacado-indicador{
  margin-top: 8px;
  font-weight: 800;
  color: #0ea5e9;
  font-size: 0.9em;
  background: rgba(14,165,233,0.12);
  border: 1px solid rgba(14,165,233,0.35);
  padding: 6px 12px;
  border-radius: 999px;
}


.carrito-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  position: relative;
}

.carrito-close {
  background: transparent !important;
  border: none !important;
  padding: 0 !important;
  line-height: 1;
  box-shadow: none;
  color: #333;
  font-size: 18px;
  cursor: pointer;

  position: absolute;
  top: -2px;
  right: 0;
}



.modal-close {
  background: transparent !important;
  border: none !important;
  padding: 0 !important;
  line-height: 1;
  box-shadow: none;
}





/* Responsive móvil: 650px hacia abajo */
@media (max-width: 430px) {
  header {
    padding: 14px 12px;
    gap: 10px;
    justify-content: center;
  }

  header img {
    height: 62px;
    width: auto;
    max-width: 85vw;
    display: block;
  }

  header .header-search {
    width: 100%;
    justify-content: center;
  }

  header .header-actions {
    width: 100%;
    display: flex;
    justify-content: center;
    gap: 10px;
  }
}

@media (max-width: 650px) {
  header {
    padding: 14px 12px;
    gap: 10px;
  }

  header img {
    height: 56px;
    width: auto;
    max-width: 85vw;
    display: block;
    margin: 0 auto;
  }

  header {
    flex-direction: column;
    gap: 10px;
    padding: 14px 12px;
    justify-content: center;
    align-items: center;
  }

  header img {
    height: 64px;
    width: auto;
    max-width: 100%;
  }



  /* Asegura centrado real incluso si otros estilos del header ponen width/margin raros */
  header .header-actions {
    width: 100%;
    display: flex;
    justify-content: center;

    gap: 10px;
  }

  header .btn-add,
  header button {
    /* Evita que el layout del menú quede muy “largo” por cada tarjeta */
    width: 46px;
    height: 42px;
    font-size: 18px;
    padding: 0;
    margin: 0;
  }
  #menu {
    padding: 28px 12px;
    gap: 14px;
  }

  .products {
    width: min(250px, 100%);
  }

  /* Modal carrito: que no quede cortado en pantallas chicas */
  .ventana-modal {
    top: 0.5%;

    min-width: 0;
    max-height: 85vh;
    padding: 16px;
    font-size: 1.05rem;
  }

  .ventana-modal h2 {
    font-size: 1.6em;
  }

  .factura button {
    font-size: 1.25em;
    padding: 18px 16px;
  }

  .total {
    font-size: 1.4em;
  }

  .carrito-table-header .col-label,
  .item-info span,
  .qty-badge {
    font-size: 1rem;
  }
}
</style>
