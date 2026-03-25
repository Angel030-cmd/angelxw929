<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PhotoShare - Publica tus Momentos</title>
    <!-- Tailwind CSS para el diseño -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Iconos de Font Awesome -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f8fafc;
        }

        .glass-nav {
            background: rgba(255, 255, 255, 0.8);
            backdrop-filter: blur(10px);
            border-bottom: 1px solid rgba(226, 232, 240, 0.8);
        }

        .photo-card {
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .photo-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1);
        }

        #drop-zone.dragover {
            border-color: #3b82f6;
            background-color: #eff6ff;
        }

        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.5);
            z-index: 50;
            align-items: center;
            justify-content: center;
        }

        .modal.active {
            display: flex;
        }
    </style>
</head>
<body>

    <!-- Navegación -->
    <nav class="glass-nav sticky top-0 z-40 w-full">
        <div class="max-w-5xl mx-auto px-4 h-16 flex items-center justify-between">
            <div class="flex items-center gap-2">
                <div class="bg-gradient-to-tr from-purple-600 to-blue-500 p-2 rounded-lg">
                    <i class="fas fa-camera text-white text-xl"></i>
                </div>
                <span class="text-xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-purple-600 to-blue-600">
                    PhotoShare
                </span>
            </div>
            <!-- Botón para abrir el modal de subida -->
            <button onclick="toggleModal(true)" class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-full font-medium transition-colors flex items-center gap-2">
                <i class="fas fa-plus"></i>
                <span class="hidden sm:inline">Publicar</span>
            </button>
        </div>
    </nav>

    <!-- Contenido Principal -->
    <main class="max-w-5xl mx-auto px-4 py-8">
        <!-- Feed de Fotos: Aquí se insertan las imágenes dinámicamente -->
        <div id="photo-feed" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"></div>

        <!-- Mensaje cuando no hay fotos -->
        <div id="empty-state" class="text-center py-20 hidden">
            <div class="text-6xl text-slate-300 mb-4">
                <i class="fas fa-images"></i>
            </div>
            <h2 class="text-2xl font-semibold text-slate-600">No hay fotos aún</h2>
            <p class="text-slate-500 mt-2">¡Sé el primero en compartir un momento!</p>
        </div>
    </main>

    <!-- Modal para Subir Foto -->
    <div id="upload-modal" class="modal px-4">
        <div class="bg-white rounded-2xl w-full max-w-md overflow-hidden shadow-2xl">
            <div class="p-6 border-b flex justify-between items-center">
                <h3 class="text-xl font-bold text-slate-800">Nueva Publicación</h3>
                <button onclick="toggleModal(false)" class="text-slate-400 hover:text-slate-600">
                    <i class="fas fa-times text-xl"></i>
                </button>
            </div>
            
            <div class="p-6">
                <!-- Zona de arrastrar y soltar archivo -->
                <div id="drop-zone" class="border-2 border-dashed border-slate-300 rounded-xl p-8 text-center cursor-pointer hover:border-blue-400 transition-colors mb-4">
                    <input type="file" id="file-input" class="hidden" accept="image/*">
                    <div id="preview-container" class="hidden mb-4">
                        <img id="image-preview" src="" alt="Preview" class="max-h-48 mx-auto rounded-lg shadow-md">
                    </div>
                    <div id="upload-prompt">
                        <i class="fas fa-cloud-upload-alt text-4xl text-blue-500 mb-3"></i>
                        <p class="text-slate-600 font-medium">Haz clic o arrastra una imagen</p>
                        <p class="text-xs text-slate-400 mt-1">PNG, JPG o GIF hasta 5MB</p>
                    </div>
                </div>

                <!-- Formulario de descripción -->
                <div class="space-y-4">
                    <div>
                        <label class="block text-sm font-semibold text-slate-700 mb-1">Descripción</label>
                        <textarea id="photo-caption" rows="3" class="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:outline-none resize-none" placeholder="¿Qué está pasando en esta foto?"></textarea>
                    </div>
                    <button id="publish-btn" class="w-full bg-blue-600 text-white py-3 rounded-xl font-bold hover:bg-blue-700 transition-colors">
                        Publicar ahora
                    </button>
                </div>
            </div>
        </div>
    </div>

    <!-- Notificación emergente (Toast) -->
    <div id="toast" class="fixed bottom-8 left-1/2 -translate-x-1/2 bg-slate-800 text-white px-6 py-3 rounded-full shadow-lg transform translate-y-20 opacity-0 transition-all duration-300 z-50">
        ¡Foto publicada con éxito!
    </div>

    <script>
        // Referencias a elementos del DOM
        const dropZone = document.getElementById('drop-zone');
        const fileInput = document.getElementById('file-input');
        const previewContainer = document.getElementById('preview-container');
        const imagePreview = document.getElementById('image-preview');
        const uploadPrompt = document.getElementById('upload-prompt');
        const publishBtn = document.getElementById('publish-btn');
        const captionInput = document.getElementById('photo-caption');
        const photoFeed = document.getElementById('photo-feed');
        const emptyState = document.getElementById('empty-state');
        const toast = document.getElementById('toast');

        let currentImageData = null;

        // Datos iniciales de ejemplo
        let photos = [
            {
                id: 1,
                url: 'https://images.unsplash.com/photo-1506744038136-46273834b3fb?auto=format&fit=crop&w=800&q=80',
                caption: 'Atardecer increíble en las montañas. #naturaleza',
                likes: 24,
                date: 'Hace 2 horas'
            },
            {
                id: 2,
                url: 'https://images.unsplash.com/photo-1470071459604-3b5ec3a7fe05?auto=format&fit=crop&w=800&q=80',
                caption: 'Explorando el bosque esta mañana.',
                likes: 12,
                date: 'Hace 5 horas'
            }
        ];

        // Función para abrir/cerrar el modal
        function toggleModal(show) {
            const modal = document.getElementById('upload-modal');
            if (show) {
                modal.classList.add('active');
            } else {
                modal.classList.remove('active');
                resetForm();
            }
        }

        // Limpiar el formulario de subida
        function resetForm() {
            fileInput.value = '';
            captionInput.value = '';
            currentImageData = null;
            previewContainer.classList.add('hidden');
            uploadPrompt.classList.remove('hidden');
        }

        // Eventos para manejo de archivos
        dropZone.addEventListener('click', () => fileInput.click());
        fileInput.addEventListener('change', handleFile);
        dropZone.addEventListener('dragover', (e) => { e.preventDefault(); dropZone.classList.add('dragover'); });
        dropZone.addEventListener('dragleave', () => { dropZone.classList.remove('dragover'); });
        dropZone.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('dragover');
            const files = e.dataTransfer.files;
            if (files.length) handleFile({ target: { files } });
        });

        // Convertir imagen a Base64 para previsualización
        function handleFile(e) {
            const file = e.target.files[0];
            if (file && file.type.startsWith('image/')) {
                const reader = new FileReader();
                reader.onload = (event) => {
                    currentImageData = event.target.result;
                    imagePreview.src = currentImageData;
                    uploadPrompt.classList.add('hidden');
                    previewContainer.classList.remove('hidden');
                };
                reader.readAsDataURL(file);
            }
        }

        // Función para añadir la foto al feed
        publishBtn.addEventListener('click', () => {
            if (!currentImageData) return;

            const newPhoto = {
                id: Date.now(),
                url: currentImageData,
                caption: captionInput.value || 'Sin descripción',
                likes: 0,
                date: 'Recién publicado'
            };

            photos.unshift(newPhoto);
            renderFeed();
            toggleModal(false);
            showToast();
        });

        // Mostrar notificación de éxito
        function showToast() {
            toast.classList.remove('translate-y-20', 'opacity-0');
            setTimeout(() => {
                toast.classList.add('translate-y-20', 'opacity-0');
            }, 3000);
        }

        // Renderizar las fotos en la pantalla
        function renderFeed() {
            if (photos.length === 0) {
                emptyState.classList.remove('hidden');
                photoFeed.innerHTML = '';
                return;
            }

            emptyState.classList.add('hidden');
            photoFeed.innerHTML = photos.map(photo => `
                <div class="photo-card bg-white rounded-2xl overflow-hidden shadow-sm border border-slate-200">
                    <img src="${photo.url}" alt="Post" class="w-full h-64 object-cover">
                    <div class="p-4">
                        <div class="flex justify-between items-center mb-3">
                            <button onclick="likePhoto(${photo.id})" class="flex items-center gap-2 text-slate-600 hover:text-red-500 transition-colors">
                                <i class="fa-heart far text-xl"></i>
                                <span class="font-semibold">${photo.likes}</span>
                            </button>
                            <span class="text-xs text-slate-400 font-medium uppercase tracking-wider">${photo.date}</span>
                        </div>
                        <p class="text-slate-700 text-sm leading-relaxed">
                            <span class="font-bold text-slate-900 mr-2">Usuario</span>${photo.caption}
                        </p>
                    </div>
                </div>
            `).join('');
        }

        // Lógica simple de Likes
        window.likePhoto = function(id) {
            const photo = photos.find(p => p.id === id);
            if (photo) {
                photo.likes++;
                renderFeed();
            }
        };

        // Carga inicial
        document.addEventListener('DOMContentLoaded', renderFeed);
    </script>
</body>
</html>
