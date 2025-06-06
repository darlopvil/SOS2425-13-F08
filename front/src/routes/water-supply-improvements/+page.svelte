<script lang="ts">
	import { onMount } from 'svelte';
	import { dev } from '$app/environment';
	import { goto } from '$app/navigation';
	import { messageStore } from '$lib/stores/messageStore';

	interface Datos {
		year: number;
		autonomous_community: string;
		amount: number;
		benefited_population: number;
		project_count: number;
	}

	let datos: Datos[] = [];
	let mensaje = '';
	let tipoMensaje: string = 'primary';
	let limit = 10;
	let offset = 0;
	const BASE_URL = dev ? 'http://localhost:16078' : 'https://sos2425-13.onrender.com';
	const API = `${BASE_URL}/api/v1/water-supply-improvements`;

	let filtrosAplicados = '';
	let query = `?limit=${limit}&offset=${offset}`;

	// Campos de formulario y búsqueda
	let year = '';
	let comunidad = '';
	let cantidad = '';
	let poblacion = '';
	let proyectos = '';

	let busquedaYear = '';
	let busquedaComunidad = '';
	let busquedaCantidad = '';
	let busquedaPoblacion = '';
	let busquedaProyectos = '';

	let desde = '';
	let hasta = '';

	let editMode = false;
	let currentEdit: Datos | null = null;

	onMount(() => {
		const unsubscribe = messageStore.subscribe((value) => {
			if (value) {
				mensaje = value.message;
				tipoMensaje = value.type;
				setTimeout(() => {
					mensaje = '';
					messageStore.set(null);
				}, 3000);
			}
		});
		obtenerDatos();
		return () => unsubscribe();
	});

	async function obtenerDatos() {
		try {
			const res = await fetch(API + query);
			if (!res.ok) {
				const err = await res.json();
				throw new Error(err.error || 'Error al obtener datos');
			}
			datos = await res.json();
			if (datos.length === 0) {
				mostrarMensaje('⚠️ No se encontraron datos que coincidan con los filtros', 'danger');
				// Redirigir a una página 404 si lo prefieres
				// goto('/404');
			}
		} catch (err) {
			mensaje = err instanceof Error ? err.message : 'Error desconocido';
			tipoMensaje = 'danger';
		}
	}

	function mostrarMensaje(texto: string, tipo: typeof tipoMensaje = 'info') {
		mensaje = texto;
		tipoMensaje = tipo;
		setTimeout(() => (mensaje = ''), 4000);
	}

	async function cargarIniciales() {
		try {
			const res = await fetch(API + '/loadInitialData');
			if (!res.ok) throw new Error();
			mostrarMensaje('✅ Datos iniciales cargados', 'success');
			await obtenerDatos();
		} catch {
			mostrarMensaje('⚠️ Ya existen datos iniciales', 'warning');
		}
	}

	function limpiarFormulario() {
		year = comunidad = cantidad = poblacion = proyectos = '';
	}

	async function crear() {
		try {
			const nuevo: Datos = {
				year: +year,
				autonomous_community: comunidad,
				amount: +cantidad,
				benefited_population: +poblacion,
				project_count: +proyectos
			};
			const res = await fetch(API, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(nuevo)
			});
			if (res.status === 201) {
				mostrarMensaje('✅ Recurso creado', 'success');
				datos = [nuevo, ...datos];
				limpiarFormulario();
			} else if (res.status === 409) mostrarMensaje('⚠️ Recurso ya existe', 'warning');
			else if (res.status === 400) mostrarMensaje('⚠️ Campos incompletos', 'warning');
			else {
				const err = await res.json();
				throw new Error(err.error);
			}
		} catch (e) {
			mostrarMensaje(e instanceof Error ? e.message : 'Error al crear', 'danger');
		}
	}

	async function eliminarTodo() {
		try {
			const res = await fetch(API, { method: 'DELETE' });
			if (!res.ok) {
				const err = await res.json();
				throw new Error(err.error);
			}
			datos = [];
			mostrarMensaje('🗑️ Todos los datos eliminados', 'success');
		} catch (e) {
			mostrarMensaje(e instanceof Error ? e.message : 'Error al eliminar', 'danger');
		}
	}

	function aplicarPaginacion() {
		query = `?limit=${limit}&offset=${offset}${filtrosAplicados}`;
	}

	function paginaAnterior() {
		if (offset >= limit) {
			offset -= limit;
			aplicarPaginacion();
			obtenerDatos();
		} else mostrarMensaje('⚠️ Primera página', 'warning');
	}

	function siguientePagina() {
		offset += limit;
		aplicarPaginacion();
		obtenerDatos();
	}

	function buildFilters() {
		const f: string[] = [];
		if (busquedaYear) f.push(`year=${busquedaYear}`);
		if (busquedaComunidad) f.push(`autonomous_community=${busquedaComunidad}`);
		if (busquedaCantidad) f.push(`amount=${busquedaCantidad}`);
		if (busquedaPoblacion) f.push(`benefited_population=${busquedaPoblacion}`);
		if (busquedaProyectos) f.push(`project_count=${busquedaProyectos}`);
		filtrosAplicados = f.length ? '&' + f.join('&') : '';
		offset = 0;
		aplicarPaginacion();
		obtenerDatos();
	}

	function buscarIntervalo() {
		const f: string[] = [];
		if (desde) f.push(`from=${desde}`);
		if (hasta) f.push(`to=${hasta}`);
		filtrosAplicados = f.length ? '&' + f.join('&') : '';
		offset = 0;
		aplicarPaginacion();
		obtenerDatos();
		mostrarMensaje('🕒 Intervalo aplicado', 'info');
	}

	function editarRecurso(r: Datos) {
		currentEdit = { ...r };
		editMode = true;
		goto(`/water-supply-improvements/${r.year}/${encodeURIComponent(r.autonomous_community)}`);
	}

	async function actualizarRecurso() {
		if (!currentEdit) return mostrarMensaje('⚠️ Ningún recurso seleccionado', 'danger');
		try {
			const res = await fetch(
				`${API}/${currentEdit.year}/${encodeURIComponent(currentEdit.autonomous_community)}`,
				{
					method: 'PUT',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify(currentEdit)
				}
			);
			if (res.ok) {
				mostrarMensaje('✅ Recurso actualizado', 'success');
				editMode = false;
				obtenerDatos();
			} else if (res.status === 409) mostrarMensaje('⚠️ Conflicto de datos', 'warning');
			else if (res.status === 400) mostrarMensaje('⚠️ Datos inválidos', 'warning');
			else throw new Error();
		} catch {
			mostrarMensaje('Error al actualizar', 'danger');
		}
	}

	async function eliminarRecurso(r: Datos) {
		try {
			const res = await fetch(`${API}/${r.year}/${encodeURIComponent(r.autonomous_community)}`, {
				method: 'DELETE'
			});
			if (res.ok) {
				datos = datos.filter(
					(d) => d.year !== r.year || d.autonomous_community !== r.autonomous_community
				);
				mostrarMensaje('🗑️ Recurso eliminado', 'success');
			} else {
				const err = await res.json();
				throw new Error(err.error);
			}
		} catch (e) {
			mostrarMensaje(e instanceof Error ? e.message : 'Error al eliminar', 'danger');
		}
	}

	function cancelarEdicion() {
		editMode = false;
		currentEdit = null;
	}
</script>

<svelte:head>
	<title>Gestión de Recursos de Abastecimiento de Agua</title>
</svelte:head>

<main class="container">
	{#if mensaje}
		<div class="alert alert-{tipoMensaje}">{mensaje}</div>
	{/if}

	<h1>Gestión de Recursos de Abastecimiento de Agua</h1>

	<div class="section card">
		<div class="flex">
			<button class="btn btn-info" on:click={cargarIniciales}>Cargar datos iniciales</button>
			<button class="btn btn-danger" on:click={eliminarTodo}>Eliminar todo</button>
		</div>
	</div>

	<div class="section card">
		<h3>Búsqueda por intervalo de años</h3>
		<div class="flex">
			<input type="number" placeholder="Desde" bind:value={desde} />
			<input type="number" placeholder="Hasta" bind:value={hasta} />
			<button class="btn btn-primary" on:click={buscarIntervalo}>Buscar</button>
		</div>
	</div>

	<div class="section card">
		<h3>Filtrado por campos</h3>
		<div class="flex">
			<input placeholder="Año" bind:value={busquedaYear} />
			<input placeholder="Comunidad Autónoma" bind:value={busquedaComunidad} />
			<input placeholder="Cantidad" bind:value={busquedaCantidad} />
			<input placeholder="Población" bind:value={busquedaPoblacion} />
			<input placeholder="Proyectos" bind:value={busquedaProyectos} />
			<button class="btn btn-warning" on:click={buildFilters}>Filtrar</button>
		</div>
	</div>

	<div class="section card">
		<h3>{editMode ? 'Editar recurso' : 'Crear nuevo recurso'}</h3>
		<div class="flex">
			<input type="number" placeholder="AñoC" bind:value={year} />
			<input placeholder="Comunidad AutónomaC" bind:value={comunidad} />
			<input type="number" step="0.01" placeholder="CantidadC (€)" bind:value={cantidad} />
			<input type="number" placeholder="Población beneficiadaC" bind:value={poblacion} />
			<input type="number" placeholder="ProyectosC" bind:value={proyectos} />
			{#if editMode}
				<button class="btn btn-success" on:click={actualizarRecurso}>Guardar</button>
				<button class="btn btn-outline" on:click={cancelarEdicion}>Cancelar</button>
			{:else}
				<button class="btn btn-success" on:click={crear}>Añadir</button>
			{/if}
		</div>
	</div>

	<div class="section card table-wrapper">
		<table>
			<thead>
				<tr>
					<th>Año</th>
					<th>Comunidad</th>
					<th>Cantidad (€)</th>
					<th>Población</th>
					<th>Proyectos</th>
					<th>Acciones</th>
				</tr>
			</thead>
			<tbody>
				{#each datos as d}
					<tr>
						<td>{d.year}</td>
						<td>{d.autonomous_community}</td>
						<td>{d.amount.toLocaleString()}</td>
						<td>{d.benefited_population.toLocaleString()}</td>
						<td>{d.project_count}</td>
						<td class="flex">
							{#if editMode && currentEdit?.year === d.year}
								<button class="btn btn-success btn-sm" on:click={actualizarRecurso}>Guardar</button>
								<button class="btn btn-outline btn-sm" on:click={cancelarEdicion}>Cancelar</button>
							{:else}
								<button class="btn btn-warning btn-sm" on:click={() => editarRecurso(d)}
									>Editar</button
								>
								<button class="btn btn-danger btn-sm" on:click={() => eliminarRecurso(d)}
									>Eliminar</button
								>
							{/if}
						</td>
					</tr>
				{/each}
			</tbody>
		</table>
	</div>

	<div class="section flex">
		<button class="btn btn-outline" on:click={paginaAnterior}>Anterior</button>
		<button class="btn btn-outline" on:click={siguientePagina}>Siguiente</button>
	</div>
</main>

<style>
	:root {
		--primary: #0056b3;
		--secondary: #6c757d;
		--bg: #f5f5f5;
		--card-bg: #ffffff;
		--border: #dee2e6;
		--radius: 6px;
		--shadow: rgba(0, 0, 0, 0.08);
		--font: 'Roboto', sans-serif;
	}
	body {
		font-family: var(--font);
		background: var(--bg);
		color: #212529;
	}
	.container {
		max-width: 900px;
		margin: 2rem auto;
		padding: 1.5rem;
		background: var(--card-bg);
		border-radius: var(--radius);
		box-shadow: 0 4px 12px var(--shadow);
	}
	h1 {
		font-size: 1.75rem;
		margin-bottom: 1rem;
		color: var(--primary);
		border-bottom: 2px solid var(--primary);
		padding-bottom: 0.5rem;
	}
	h3 {
		font-size: 1.25rem;
		margin-top: 1.5rem;
		margin-bottom: 0.75rem;
		color: var(--secondary);
	}
	.flex {
		display: flex;
		flex-wrap: wrap;
		gap: 0.75rem;
		align-items: center;
	}
	.section {
		margin-bottom: 1.5rem;
	}
	.card {
		background: var(--card-bg);
		border: 1px solid var(--border);
		border-radius: var(--radius);
		padding: 1rem;
		box-shadow: 0 2px 6px var(--shadow);
	}
	.table-wrapper {
		overflow-x: auto;
		margin-top: 1rem;
	}
	table {
		width: 100%;
		border-collapse: collapse;
	}
	th,
	td {
		padding: 0.75rem 0.5rem;
		border: 1px solid var(--border);
	}
	th {
		background: var(--bg);
		position: sticky;
		top: 0;
	}
	tr:nth-child(even) {
		background: #fafafa;
	}
	tr:hover {
		background: #f1f1f1;
	}
	input {
		flex: 1;
		min-width: 120px;
		padding: 0.5rem;
		border: 1px solid var(--border);
		border-radius: var(--radius);
	}
	.btn {
		padding: 0.5rem 1rem;
		border: none;
		border-radius: var(--radius);
		cursor: pointer;
		transition: transform 0.1s ease;
	}
	.btn:hover {
		transform: translateY(-1px);
	}
	.btn:active {
		transform: translateY(0);
	}
	.btn-primary {
		background: var(--primary);
		color: white;
	}
	.btn-success {
		background: #28a745;
		color: white;
	}
	.btn-info {
		background: #17a2b8;
		color: white;
	}
	.btn-warning {
		background: #ffc107;
		color: #212529;
	}
	.btn-danger {
		background: #dc3545;
		color: white;
	}
	.btn-outline {
		background: transparent;
		border: 1px solid var(--primary);
		color: var(--primary);
	}
	.btn-sm {
		padding: 0.4rem 0.75rem;
		font-size: 0.85rem;
	}
	.alert {
		padding: 0.75rem 1rem;
		border-radius: var(--radius);
		margin-bottom: 1rem;
		font-weight: 500;
	}
	.alert-primary {
		background: rgba(0, 86, 179, 0.1);
		color: var(--primary);
	}
	.alert-success {
		background: rgba(40, 167, 69, 0.1);
		color: #28a745;
	}
	.alert-info {
		background: rgba(23, 162, 184, 0.1);
		color: #17a2b8;
	}
	.alert-warning {
		background: rgba(255, 193, 7, 0.1);
		color: #856404;
	}
	.alert-danger {
		background: rgba(220, 53, 69, 0.1);
		color: #dc3545;
	}
	@media (max-width: 600px) {
		.flex {
			flex-direction: column;
		}
	}
</style>
