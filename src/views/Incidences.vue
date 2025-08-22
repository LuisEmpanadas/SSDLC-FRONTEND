<template>
  <div class="page">
    <!-- Sección para crear un nuevo incidente -->
    <div class="new-incidence">
      <h2>Nuevo Incidente</h2>
      <form @submit.prevent="addIncidence">
        <input v-model="newIncidence.type" placeholder="Tipo" required />
        <input v-model="newIncidence.location" placeholder="Ubicación" required />
        <textarea v-model="newIncidence.description" placeholder="Descripción" required></textarea>
        <button type="submit">Agregar</button>
      </form>
    </div>

    <!-- Sección de incidencias -->
    <div class="incidence-list">
      <div class="header-section">
        <h2>Mis Incidencias</h2>
        <button @click="logout" class="logout-button">Cerrar Sesión</button>
      </div>

      <div v-if="incidences.length" class="incidences-container">
        <div v-for="i in incidences" :key="i.id" class="incidence-item">
          <h3>
            {{ i.type }}
            <span class="status" :class="i.status">{{ i.status }}</span>
          </h3>
          <p><strong>Ubicación:</strong> {{ i.location }}</p>
          <p>{{ i.description }}</p>
          <p v-if="i.attachment">
            <a :href="getFileUrl(i.attachment)" target="_blank">Ver adjunto</a>
          </p>
          <small>Registrado el: {{ formateDate(i.created_at) }}</small>
        </div>
      </div>
      <p v-else class="no-data">No hay incidencias registradas.</p>
    </div>

    <!-- Sección de reportes exportables -->
    <div class="reports-export">
      <h2>Reportes Exportables</h2>
      <div class="export-buttons">
        <button @click="exportPDF" class="export-btn pdf">
          <span class="btn-icon">📄</span> Exportar PDF
        </button>
        <button @click="exportExcel" class="export-btn excel">
          <span class="btn-icon">📊</span> Exportar Excel
        </button>
      </div>

      <div class="table-container">
        <table v-if="incidences.length" class="reports-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Tipo</th>
              <th>Ubicación</th>
              <th>Descripción</th>
              <th>Estado</th>
              <th>Fecha</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="r in incidences" :key="r.id">
              <td>{{ r.id }}</td>
              <td>{{ r.type }}</td>
              <td>{{ r.location }}</td>
              <td>{{ r.description }}</td>
              <td>
                <span class="table-status" :class="r.status">{{ r.status }}</span>
              </td>
              <td>{{ formateDate(r.created_at) }}</td>
            </tr>
          </tbody>
        </table>
        <p v-else class="no-data">No hay reportes para exportar.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref, onMounted } from 'vue'
import axios from '@/api/axios'
import { useUserStore } from '@/stores/user'
import { useRouter } from 'vue-router'
import jsPDF from 'jspdf'
import autoTable from 'jspdf-autotable' // <-- importante
import * as XLSX from 'xlsx'

const incidences = ref([])
const newIncidence = reactive({ type: '', location: '', description: '' })
const archivo = ref(null)

const userStore = useUserStore()
const router = useRouter()

// Obtener incidencias al montar
onMounted(async () => {
  try {
    const res = await axios.get('/api/reports')
    incidences.value = res.data
  } catch (error) {
    console.error('Error al obtener reportes:', error)
    router.push('/login')
  }
})

// Manejar cambio de archivo
const onFileChange = (e) => {
  archivo.value = e.target.files[0]
}

// Guardar incidencia en Laravel
const addIncidence = async () => {
  try {
    await axios.get('/sanctum/csrf-cookie')

    const formData = new FormData()
    formData.append('type', newIncidence.type)
    formData.append('location', newIncidence.location)
    formData.append('description', newIncidence.description)
    if (archivo.value) formData.append('attachment', archivo.value)

    const res = await axios.post('/api/reports', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    })

    incidences.value.push(res.data)
    newIncidence.type = ''
    newIncidence.location = ''
    newIncidence.description = ''
    archivo.value = null
  } catch (error) {
    console.error('Error al guardar incidencia:', error.response || error)
  }
}

// Cerrar sesión
const logout = async () => {
  await userStore.logout()
  router.push('/login')
}

// Obtener URL de archivo
const getFileUrl = (path) => `http://127.0.0.1:8000/storage/${path}`

// Formatear fecha
const formateDate = (datetime) => new Date(datetime).toLocaleDateString()

// Exportar PDF con jsPDF + autoTable
const exportPDF = () => {
  const doc = new jsPDF()
  doc.setFontSize(18)
  doc.setTextColor(55, 65, 81)
  doc.text('Reporte de Incidencias', 14, 20)

  const tableColumn = ['ID', 'Tipo', 'Ubicación', 'Descripción', 'Estado', 'Fecha']
  const tableRows = incidences.value.map((r) => [
    r.id,
    r.type,
    r.location,
    r.description,
    r.status.charAt(0).toUpperCase() + r.status.slice(1),
    formateDate(r.created_at),
  ])

  autoTable(doc, {
    head: [tableColumn],
    body: tableRows,
    startY: 30,
    theme: 'striped',
    headStyles: { fillColor: [59, 130, 246], textColor: 255, fontStyle: 'bold' },
    bodyStyles: { fontSize: 10, textColor: [55, 65, 81] },
    alternateRowStyles: { fillColor: [243, 244, 246] },
  })

  doc.save('reporte_incidencias.pdf')
}

// Exportar Excel
const exportExcel = () => {
  const tableData = incidences.value.map((r) => ({
    ID: r.id,
    Tipo: r.type,
    Ubicación: r.location,
    Descripción: r.description,
    Estado: r.status,
    Fecha: formateDate(r.created_at),
  }))
  const worksheet = XLSX.utils.json_to_sheet(tableData)
  const workbook = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(workbook, worksheet, 'Reportes')
  XLSX.writeFile(workbook, 'reporte_incidencias.xlsx')
}
</script>

<style scoped>
/* Variables de colores */
:root {
  --primary: #4361ee;
  --secondary: #3a0ca3;
  --accent: #f72585;
  --success: #4cc9f0;
  --warning: #f9c74f;
  --danger: #f94144;
  --light: #f8f9fa;
  --dark: #212529;
  --gray: #6c757d;
  --light-gray: #e9ecef;
}

.page {
  max-width: auto;
  margin-left: 300px;
  padding: 2rem 1rem;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
  min-height: 100vh;
}

/* Nuevo incidente */
.new-incidence {
  background: #000000;
  padding: 2rem;
  margin-bottom: 2rem;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.new-incidence::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 5px;
  height: 100%;
  background: linear-gradient(to bottom, var(--primary), var(--secondary));
}

.new-incidence:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.15);
}

.new-incidence h2 {
  margin-bottom: 1.5rem;
  color: var(--dark);
  font-size: 1.8rem;
  position: relative;
  display: inline-block;
}

.new-incidence h2::after {
  content: '';
  position: absolute;
  bottom: -8px;
  left: 0;
  width: 50px;
  height: 4px;
  background: var(--accent);
  border-radius: 2px;
}

.new-incidence input,
.new-incidence textarea {
  width: 100%;
  margin-bottom: 1.2rem;
  padding: 0.9rem 1.2rem;
  border-radius: 10px;
  border: 2px solid var(--light-gray);
  transition: all 0.3s ease;
  font-size: 1rem;
}

.new-incidence input:focus,
.new-incidence textarea:focus {
  border-color: var(--primary);
  outline: none;
  box-shadow: 0 0 0 3px rgba(67, 97, 238, 0.2);
  transform: scale(1.01);
}

.new-incidence textarea {
  min-height: 100px;
  resize: vertical;
}

.new-incidence button {
  padding: 0.9rem 1.8rem;
  background: linear-gradient(to right, var(--primary), var(--secondary));
  color: white;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-weight: 600;
  font-size: 1rem;
  transition: all 0.3s ease;
  letter-spacing: 0.5px;
  text-transform: uppercase;
  position: relative;
  overflow: hidden;
}

.new-incidence button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: all 0.5s ease;
}

.new-incidence button:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(67, 97, 238, 0.4);
}

.new-incidence button:hover::before {
  left: 100%;
}

/* Lista de incidencias */
.incidence-list {
  background: #000000;
  padding: 2rem;
  margin-bottom: 2rem;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.header-section h2 {
  color: var(--dark);
  font-size: 1.8rem;
  margin: 0;
}

.logout-button {
  background: linear-gradient(to right, var(--danger), #d90429);
  color: white;
  padding: 0.7rem 1.5rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.logout-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(249, 65, 68, 0.4);
}

.incidences-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.incidence-item {
  background: #ffffff;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
  border-left: 5px solid var(--primary);
  position: relative;
  overflow: hidden;
}

.incidence-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.incidence-item h3 {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0;
  color: black;
  font-size: 1.2rem;
}

.status {
  font-size: 0.75rem;
  padding: 0.35rem 0.8rem;
  border-radius: 20px;
  text-transform: uppercase;
  font-weight: 700;
  letter-spacing: 0.5px;
}

.status.pending {
  background: #fff3cd;
  color: #856404;
}

.status.in_review {
  background: #cce7ff;
  color: #004085;
}

.status.resolved {
  background: #d4edda;
  color: #155724;
}

.incidence-item p {
  color: black;
  margin: 0.8rem 0;
  line-height: 1.5;
}

.incidence-item a {
  color: black;
  text-decoration: none;
  font-weight: 600;
  transition: all 0.2s ease;
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
}

.incidence-item a:hover {
  color: rgb(0, 0, 0);
  text-decoration: underline;
}

.incidence-item small {
  color: rgb(0, 0, 0);
  font-size: 0.85rem;
}

/* Reportes exportables */
.reports-export {
  background: #000000;
  padding: 2rem;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.reports-export h2 {
  font-size: 1.8rem;
  margin-bottom: 1.5rem;
}

.export-buttons {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
}

.export-btn {
  padding: 0.9rem 1.5rem;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.export-btn.pdf {
  background: linear-gradient(to right, #e50e0e, #ed0000);
  color: white;
}

.export-btn.excel {
  background: linear-gradient(to right, #0a7c4d, #0da86d);
  color: white;
}

.export-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(0, 0, 0, 0.2);
}

.table-container {
  overflow-x: auto;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
}

.reports-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  border-radius: 10px;
  overflow: hidden;
}

.reports-table th {
  background: linear-gradient(to right, var(--primary), var(--secondary));
  color: white;
  padding: 1rem;
  text-align: left;
  font-weight: 600;
  position: sticky;
  top: 0;
}

.reports-table td {
  padding: 1rem;
  border-bottom: 1px solid var(--light-gray);
  color: rgb(255, 255, 255);
}

.reports-table tr:nth-child(even) {
  background-color: #393939;
}

.reports-table tr:hover {
  background-color: #e9ecef;
  transition: background-color 0.2s ease;
}

.table-status {
  padding: 0.35rem 0.8rem;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
}

.table-status.pending {
  background: #fff3cd;
  color: #856404;
}

.table-status.in_review {
  background: #b1d9fc;
  color: #004085;
}

.table-status.resolved {
  background: #d4edda;
  color: #155724;
}

.no-data {
  text-align: center;
  padding: 2rem;
  color: var(--gray);
  font-style: italic;
}

/* Responsive */
@media (max-width: 768px) {
  .page {
    padding: 1rem 0.5rem;
  }

  .new-incidence,
  .incidence-list,
  .reports-export {
    padding: 1.5rem;
  }

  .header-section {
    flex-direction: column;
    align-items: flex-start;
  }

  .incidences-container {
    grid-template-columns: 1fr;
  }

  .export-buttons {
    flex-direction: column;
  }

  .export-btn {
    width: 100%;
    justify-content: center;
  }

  .reports-table {
    font-size: 0.85rem;
  }

  .reports-table th,
  .reports-table td {
    padding: 0.7rem;
  }
}

@media (max-width: 480px) {
  .new-incidence h2,
  .incidence-list h2,
  .reports-export h2 {
    font-size: 1.5rem;
  }

  .new-incidence input,
  .new-incidence textarea {
    padding: 0.7rem 1rem;
  }

  .incidence-item {
    background-color: #004085;
    padding: 1.2rem;
  }
}
</style>
