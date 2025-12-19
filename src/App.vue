<script setup>
import { ref, reactive, onMounted } from 'vue'

// API base: set VITE_API_URL in your Vite env or default to wrangler dev host
const API_BASE = import.meta.env.VITE_API_URL

const items = ref([])
const showList = ref(true)
const loading = ref(false)
const message = ref('')

// Search
const searchId = ref('')
const searchResult = ref(null)

// Create
const newItem = reactive({ title: '', description: '' })

// Update
const updateId = ref('')
const updateField = ref('title')
const updateValue = ref('')

// Delete
const deleteId = ref('')

const fetchAll = async () => {
	loading.value = true
	message.value = ''
	try {
		const res = await fetch(`${API_BASE}/items`)
		items.value = await res.json()
	} catch (e) {
		message.value = 'Fetch error: ' + e.message
	} finally {
		loading.value = false
	}
}

const fetchById = async () => {
	if (!searchId.value) return (message.value = 'Enter id')
	message.value = ''
	try {
		const res = await fetch(`${API_BASE}/items/${encodeURIComponent(searchId.value)}`)
		if (res.status === 404) return (searchResult.value = null, message.value = 'Not found')
		searchResult.value = await res.json()
	} catch (e) {
		message.value = 'Fetch error: ' + e.message
	}
}

const createItem = async () => {
	if (!newItem.title) return (message.value = 'Title required')
	message.value = ''
	try {
		const res = await fetch(`${API_BASE}/items`, {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify(newItem),
		})
		if (res.status >= 400) {
			const j = await res.json().catch(() => ({}))
			return (message.value = j.error || 'Create failed')
		}
		const created = await res.json()
		message.value = 'Created id ' + (created?.id || '(unknown)')
		newItem.title = ''
		newItem.description = ''
		await fetchAll()
	} catch (e) {
		message.value = 'Create error: ' + e.message
	}
}

const updateItem = async () => {
	if (!updateId.value) return (message.value = 'Enter id')
	if (!updateValue.value) return (message.value = 'Enter value')
	message.value = ''
	try {
		// build body depending on selected field (send only the selected field)
		let body = {}
		if (updateField.value === 'title') body = { title: updateValue.value }
		else body = { description: updateValue.value }

		const res = await fetch(`${API_BASE}/items/${encodeURIComponent(updateId.value)}`, {
			method: 'PUT',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify(body),
		})
		if (res.status === 404) return (message.value = 'Not found')
		if (res.status >= 400) {
			const j = await res.json().catch(() => ({}))
			return (message.value = j.error || 'Update failed')
		}
		const updated = await res.json()
		message.value = 'Updated id ' + (updated?.id || updateId.value)
		updateId.value = ''
		updateValue.value = ''
		await fetchAll()
	} catch (e) {
		message.value = 'Update error: ' + e.message
	}
}

const deleteItem = async () => {
	if (!deleteId.value) return (message.value = 'Enter id')
	message.value = ''
	try {
		const res = await fetch(`${API_BASE}/items/${encodeURIComponent(deleteId.value)}`, { method: 'DELETE' })
		if (res.status === 404) return (message.value = 'Not found')
		if (res.status >= 400) return (message.value = 'Delete failed')
		message.value = 'Deleted id ' + deleteId.value
		deleteId.value = ''
		await fetchAll()
	} catch (e) {
		message.value = 'Delete error: ' + e.message
	}
}

onMounted(() => {
	fetchAll()
})
</script>

<template>
	<div class="container">
		<h1>CRUD Playground</h1>

		<section class="card">
			<h2 @click="showList = !showList" style="cursor:pointer">List items <small>click to toggle</small></h2>
			<div v-if="showList">
				<div class="controls">
					<button @click="fetchAll" :disabled="loading">Fetch</button>
					<span v-if="loading">Loading...</span>
				</div>
				<table class="items">
					<thead>
						<tr><th>id</th><th>title</th><th>description</th><th>created_at</th><th>updated_at</th></tr>
					</thead>
					<tbody>
						<tr v-for="it in items" :key="it.id">
							<td>{{ it.id }}</td>
							<td>{{ it.title }}</td>
							<td>{{ it.description }}</td>
							<td>{{ it.created_at }}</td>
							<td>{{ it.updated_at }}</td>
						</tr>
					</tbody>
				</table>
			</div>
		</section>

		<section class="card">
			<h2>Search by ID</h2>
			<div class="row">
				<input v-model="searchId" placeholder="id" />
				<button @click="fetchById">Search</button>
			</div>
			<pre v-if="searchResult">{{ JSON.stringify(searchResult, null, 2) }}</pre>
		</section>

		<section class="card">
			<h2>Create item</h2>
			<div class="row">
				<input v-model="newItem.title" placeholder="title" />
				<input v-model="newItem.description" placeholder="description" />
				<button @click="createItem">Create</button>
			</div>
		</section>

		<section class="card">
			<h2>Update item</h2>
			<div class="row">
				<input v-model="updateId" placeholder="id" />
				<select v-model="updateField">
					<option value="title">title</option>
					<option value="description">description</option>
				</select>
				<input v-model="updateValue" placeholder="new value" />
				<button @click="updateItem">Update</button>
			</div>
		</section>

		<section class="card">
			<h2>Delete item</h2>
			<div class="row">
				<input v-model="deleteId" placeholder="id" />
				<button @click="deleteItem">Delete</button>
			</div>
		</section>

		<div class="status" v-if="message">{{ message }}</div>
	</div>
</template>

<style scoped>
.container{max-width:900px;margin:20px auto;font-family:system-ui,Segoe UI,Roboto,Arial}
.muted{color:#666;font-size:0.9rem}
.card{border:1px solid #e2e8f0;padding:12px;border-radius:6px;margin-bottom:12px}
.row{display:flex;gap:8px;align-items:center;margin-top:8px}
input,select{padding:6px 8px;border:1px solid #cbd5e1;border-radius:4px}
button{padding:6px 10px;border-radius:4px;border:1px solid #94a3b8;background:#f8fafc;cursor:pointer}
.items{width:100%;border-collapse:collapse;margin-top:8px}
.items th,.items td{border:1px solid #e2e8f0;padding:6px;text-align:left}
.status{margin-top:10px;color:#0f172a}
h1{margin-bottom:4px}
h2{margin:0;font-size:1.05rem}
</style>
