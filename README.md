# Memoria Técnica: Infraestructura Backend, Despliegue Docker y Configuración de WordPress

## 1. Infraestructura Backend y Orquestación (Docker Compose)

El proyecto se despliega utilizando **Docker** y **Docker Compose**, lo que garantiza un entorno de desarrollo aislado, reproducible y fácil de mantener. La arquitectura consta de tres contenedores interconectados mediante una red virtual privada en modo *bridge*.

### 1.1 Diagrama de Arquitectura de Contenedores

```text
[ NAVEGADOR HOST ]
   │  ┌──────────── Puerto 8080 ────────────┐
   │  │                                     ▼
   │  │                 ┌───────────────────────────────────────┐
   │  │                 │ Contenedor: wp${PROYECTO_ID}          │
   │  │                 │ Imagen: wordpress:6.9.4-php8.5-apache │
   │  │                 └───────────────────┬───────────────────┘
   │  │                                     │
   │  │                                 wp_network (Bridge)
   │  │                                     │
   │  │                 ┌───────────────────┴───────────────────┐
   │  ├─────────────►   │ Contenedor: db${PROYECTO_ID}          │
   │  │                 │ Imagen: mariadb:11.4                 │
   │  │                 └───────────────────▲───────────────────┘
   │  │                                     │
   │  │                                 wp_network (Bridge)
   │  │                                     │
   │  │                 ┌───────────────────┴───────────────────┐
   │  └──── Puerto 8081─► Contenedor: pma_${PROYECTO_ID}        │
   │                    │ Imagen: phpmyadmin:5.2-apache         │
   │                    └───────────────────────────────────────┘
```

### 1.2 Archivo de Orquestación (`docker-compose.yml`)

```yaml
services:
  pr2_wordpress:
    image: wordpress:6.9.4-php8.5-apache
    container_name: wp${PROYECTO_ID}
    ports:
      - "8080:80"
    depends_on:
      - pr2_bd_wp
    environment:
      WORDPRESS_DB_HOST: pr2_bd_wp
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_HOME', 'http://wordpress.ivan:8080');
        define('WP_SITEURL', 'http://wordpress.ivan:8080');
    extra_hosts:
      - "wordpress.ivan:host-gateway"
    volumes:
      - wp_content:/var/www/html/wp-content   # Directorio de Wordpress
      - ./vhost.conf:/etc/apache2/sites-enabled/000-default.conf
    restart: always
    networks:
      - wp_network

  pr2_bd_wp:
    image: mariadb:11.4
    container_name: db${PROYECTO_ID}
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - bd_datos:/var/lib/mysql   # Directorio de los archivos de la BD
    restart: always
    networks:
      - wp_network

  pr2_phpmyadmin:
    image: phpmyadmin:5.2-apache
    container_name: pma_${PROYECTO_ID}
    ports:
      - "8081:80"
    environment:
      PMA_HOST: pr2_bd_wp   # Servicio de la BD
    depends_on:
      - pr2_bd_wp
    restart: always
    networks:
      - wp_network

volumes:
  bd_datos:
  wp_content:

networks:
  wp_network:
    driver: bridge
```

### 1.3 Configuración del Servidor Web (`vhost.conf`)

Para garantizar que el servidor Apache dentro del contenedor enlace correctamente las peticiones dirigidas al nombre de dominio personalizado `wordpress.ivan` en el puerto `8080`, se monta un archivo VirtualHost personalizado sobre el archivo por defecto `/etc/apache2/sites-enabled/000-default.conf`:

```apache
<VirtualHost *:80>
    ServerName wordpress.ivan
    ServerAlias www.wordpress.ivan
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

---

## 2. Configuración de WordPress y Entorno de Aplicación

### 2.1 Tema y Estilos
* **Tema Principal**: Astra.
* **Paleta de Colores Náutica**:
  * Azul Primario Profundo: `#0F2C59`
  * Turquesa/Acento: `#00A8CC`
  * Blanco/Gris Claro: `#F8FAFC`
* **Maquetador Visual**: Elementor (configurado en *Ancho Completo* para evitar barras laterales no deseadas en las páginas principales).

---

### 2.2 Plugins Instalados y Funcionalidad

| Plugin | Función Principal | Configuración / Uso en el Sitio |
| :--- | :--- | :--- |
| **Custom Post Type UI (CPT UI)** | Creación de Tipos de Contenido Personalizados | Creación del CPT `Platos` y la taxonomía `singluten`. |
| **Elementor** | Maquetación visual de páginas | Maquetación de la página *Sobre Nosotros* e integración de bloques. |
| **WPForms** | Creación de formularios | Formulario de contacto con campos de Nombre, Email, Asunto y Mensaje. |
| **Five Star Restaurant Reservations** | Gestión de reservas en línea | Sistema interactivo de reservas integrado en la página *Reservas*. |
| **Smush** | Optimización de imágenes | Compresión automática de imágenes de la lonja/mariscos para acelerar la carga. |
| **GTranslate** | Multilingüismo | Selector de idioma (Español / Inglés / Valenciano). |

---

### 2.3 Estructura de Páginas

1. **Inicio (`/`)**: Página principal maquetada con Elementor y llamadas a la acción (*CTA*) hacia reservas y carta.
2. **Carta (`/carta`)**: Muestrario de los platos del menú alimentado desde el CPT `Platos`.
3. **Sobre Nosotros (`/sobre-nosotros`)**: Historia de la marisquería en Valencia, compromiso con el producto de lonja local y opciones sin gluten.
4. **Reservas (`/reservas`)**: Formulario interactivo para la reserva de mesas procesado por *Five Star Restaurant Reservations*.
5. **Noticias / Blog (`/noticias`)**: Blog del restaurante con actualizaciones sobre productos de temporada (e.g., temporada de la clóchina valenciana).
6. **Contacto (`/contacto`)**: Información de ubicación, horarios de apertura y formulario procesado con *WPForms*.

---

### 2.4 Custom Post Types (CPT) y Taxonomías

* **Custom Post Type**: `Platos`
  * *Entradas destacadas cargadas*:
    1. **Clóchinas Valencianas al Vapor**: Cocidas al vapor con limón y pimienta (Etiqueta: `singluten`).
    2. **Gamba Roja de Cullera**: Elaborada a la sal marina (Etiqueta: `singluten`).
    3. **Arroz del Senyoret**: Arroz marinero con todo el marisco pelado (Etiqueta: `singluten`).

---

### 2.5 Usuarios y Roles del Sistema

| Usuario | Rol | Dirección de Email | Descripción / Propósito |
| :--- | :--- | :--- | :--- |
| `ivan_admin` | Administrador | `admin@wordpress.ivan` | Usuario principal de gestión del sitio y Docker. |
| `chef_valencia` | Editor | `cocina@wordpress.ivan` | Encargado de actualizar la carta y la gestión de platos (CPT). |
| `camarero_reservas` | Autor | `reservas@wordpress.ivan` | Encargado de la revisión de reservas y contacto con clientes. |

---

### 2.6 Integración RSS

* **Feed RSS Propio**:
  * **URL del Feed**: `http://wordpress.ivan:8080/wp_ivan/feed/`
  * **Ubicación**: Enlace personalizado e icono social añadido en el pie de página (*Footer Widget*).
* **Feeds RSS Externos (Barra Lateral / Sidebar)**:
  1. **Velocidad Cuchara**: `https://www.velocidadcuchara.com/feed/`
  2. **Directo al Paladar**: `https://www.directoalpaladar.com/feed`


  ---

### 2.7 Localización e Integración de Mapas

Para la página de **Contacto**, se integró un mapa interactivo mediante **Google Maps** incrustando el código `<iframe>` de la ubicación del restaurante en el centro de Valencia (Plaza del Ayuntamiento / Ciutat Vella):

* **Ubicación configurada:** Centro de Valencia (46002 Valencia, España).
* **Método de integración:** Bloque HTML personalizado / Elementor HTML con marco responsivo (`width="100%" height="450"`).

---

### 2.8 Estrategia de Copias de Seguridad y Migración (UpdraftPlus)

#### 2.8.1 Copia de Seguridad Completa
1. **Instalación de Plugin:** Se instaló y activó el plugin **UpdraftPlus Backup/Restore**.
2. **Respaldo en la Nube:** Se vinculó con **Google Drive** mediante autenticación OAuth2 y se ejecutó un respaldo completo manual incluyendo:
   * Base de datos MySQL/MariaDB.
   * Archivos de WordPress (`plugins`, `themes`, `uploads` y contenido de `wp-content`).
3. **Descarga de Artefactos:** Se descargaron localmente los ficheros comprimidos `.zip` generados por el backup.

#### 2.8.2 Restauración y Exportación
* **Restauración en Entornos:** Se reinstaló UpdraftPlus en el entorno secundario (Windows / servidor alternativo) e importaron los archivos `.zip` para ejecutar la restauración completa del sitio.
* **Exportación XML:** Se generó una copia ligera de contenidos desde **Herramientas → Exportar** seleccionando *Todo el contenido* para obtener el archivo de migración `.xml` compatible con WordPress.com.