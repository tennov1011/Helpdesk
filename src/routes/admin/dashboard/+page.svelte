<script>
	import TicketList from '$lib/component/TicketList.svelte';
	import FeedbackList from '$lib/component/FeedbackList.svelte';
	import TicketStats from '$lib/component/TicketStats.svelte';
	import TicketingForm from '$lib/component/TicketingForm.svelte';
	import { fetchEmployees } from '$lib/services/employee.js';
	import { onMount, onDestroy } from 'svelte';
	import axios from 'axios';
	import { goto } from '$app/navigation';
	import { isAuthenticated, userRole, userDepartment, userEmail } from '$lib/services/firebaseConfig';

	let tickets = [];
	let feedbacks = [];
	let employees = [];
	let authReady = false;
	let roleReady = false;
	let currentRole = undefined;
	let isLoggedIn = false;
	let adminDepartment;
	let unsubAuth, unsubRole, unsubDept;
	let showTicketModal = false;

	export let myEmployee;

	// Subscribe to user department store to track admin's department
	userDepartment.subscribe((dept) => {
		adminDepartment = dept;
	});

	// Fetch all tickets from Directus API
	async function fetchTickets() {
		try {
			const res = await axios.get('https://directus.eltamaprimaindo.com/items/TicketForm', {
				headers: {
					Authorization: `Bearer JaXaSE93k24zq7T2-vZyu3lgNOUgP8fz`
				}
			});
			tickets = res.data.data
				.sort((a, b) => new Date(b.date_created) - new Date(a.date_created))
				.map((t) => ({
					id: `TK${t.id}`,
					rawId: t.id,
					date: t.date_created,
					name: t.name,
					email: t.email,
					division: t.division,
					contactPhone: t.contactPhone,
					category: t.category,
					photo_ticket: t.photo_ticket,
					ticket: t.ticket,
					desc: t.desc,
					device: t.device,
					url_name_app: t.url_name_app,
					priority: t.priority,
					browser: t.browser,
					label: t.label,
					location: t.location,
					problem_type: t.problem_type,
					app_type: t.app_type,
					status: t.status,
					pic: t.pic,
					department: t.target_department,
					departure_time: t.departure_time,
					estimated_return_time: t.estimated_return_time
				}));
		} catch (e) {
			console.error('Gagal mengambil tiket:', e);
		}
	}

	// Fetch all feedback from Directus API
	async function fetchFeedback() {
		try {
			const res = await axios.get('https://directus.eltamaprimaindo.com/items/FeedbackForm', {
				headers: {
					Authorization: `Bearer JaXaSE93k24zq7T2-vZyu3lgNOUgP8fz`
				}
			});
			feedbacks = res.data.data
				.sort((a, b) => new Date(b.date_created) - new Date(a.date_created))
				.map((fb, idx) => ({
					id: `F${String(fb.id).padStart(3, '0')}`,
					rawId: fb.id, // Tambahkan rawId untuk referensi
					date: fb.date_created ? fb.date_created.slice(0, 10) : '',
					text: fb.feedback,
					url: fb.url,
					division: fb.division,
					name: fb.name,
					email: fb.email,
					file: fb.photo_feedback || '',
					photo_feedback: fb.photo_feedback,
					status: fb.status
				}));
		} catch (e) {
			console.error('Gagal mengambil feedback:', e);
		}
	}

	// Buka modal form tiket baru
	function openTicketModal() {
		showTicketModal = true;
	}
	// Tutup modal form tiket
	function closeTicketModal() {
		showTicketModal = false;
		selectedTicket = null; // Reset selected ticket
	}

	// Handler setelah tiket berhasil di-submit
	function handleTicketSubmitted() {
		closeTicketModal();
		fetchTickets(); // refresh data tiket
	}

	// Modal image state and handlers for ticket attachments
	let showImageModalTicket = false;
	let imageUrlTicket = '';
	// Open image modal for ticket attachment
	function handleOpenImageTicket(e) {
		imageUrlTicket = e.detail.url;
		showImageModalTicket = true;
	}
	// Close ticket image modal
	function closeImageTicket() {
		showImageModalTicket = false;
		imageUrlTicket = '';
	}

	// Modal image state and handlers for feedback attachments
	let showImageModalFeedback = false;
	let imageUrlFeedback = '';
	// Open image modal for feedback attachment
	function handleOpenImageFeedback(e) {
		imageUrlFeedback = e.detail.url;
		showImageModalFeedback = true;
	}
	// Close feedback image modal
	function closeImageFeedback() {
		showImageModalFeedback = false;
		imageUrlFeedback = '';
	}

	// Modal state for feedback detail view
	let showDetailModalFeedback = false;
	let detailTextFeedback = '';

	// Open feedback detail modal
	function handleOpenDetailFeedback(e) {
		const { feedback, photo_feedback, url } = e.detail;
		
		// Format konten untuk grid layout
		let detailContent = `Feedback:\n${feedback || 'Tidak Ada Feedback'}`;
		
		// Tambahkan URL dengan format yang sesuai untuk pemisahan di grid
		detailContent += `\n\nURL:\n${url || 'Tidak ada URL'}`;
		
		// Simpan untuk ditampilkan
		detailTextFeedback = detailContent;
		
		// Simpan URL gambar jika ada foto, atau set nilai untuk menunjukkan tidak ada lampiran
		if (photo_feedback) {
			imageUrlFeedback = `https://directus.eltamaprimaindo.com/assets/${photo_feedback}`;
		} else {
			imageUrlFeedback = ''; // Kosong untuk keperluan kondisi di tampilan
		}
		
		// Tampilkan modal detail
		showDetailModalFeedback = true;
	}
	// Close feedback detail modal
	function closeDetailModalFeedback() {
		showDetailModalFeedback = false;
		detailTextFeedback = '';
	}

	// Modal state for ticket detail view
	let showDetailModalTicket = false;
	let selectedTicket = null;
	let detailFieldsTicket = [];
	let selectedPic = '';

	// Open ticket detail modal
	function handleOpenDetailTicket(e) {
		selectedTicket = e.detail.ticket;
		detailFieldsTicket = e.detail.detailFields;
		selectedPic = e.detail.ticket.pic || '';
		showDetailModalTicket = true;
	}
	// Close ticket detail modal
	function closeDetailModalTicket() {
		showDetailModalTicket = false;
		selectedTicket = null;
		detailFieldsTicket = [];
		selectedPic = '';
	}

	// Modal state for ticket detail information text
	let showTicketDetailModal = false;
	let ticketDetailText = '';
	// Open modal to display ticket detail text
	function openTicketDetailModal(text) {
		ticketDetailText = text;
		showTicketDetailModal = true;
	}
	// Close ticket detail information modal
	function closeTicketDetailModal() {
		showTicketDetailModal = false;
		ticketDetailText = '';
	}

	// Handle ticket update event from child component
	function handleTicketUpdated(e) {
		const updated = e.detail.updatedTicket;
		tickets = tickets.map((t) => (t.rawId === updated.rawId ? { ...t, ...updated } : t));
	}

	// Delete selected ticket from Directus API
	async function deleteTicket() {
		// Check if ticket is selected
		if (!selectedTicket) return;
		// Confirm deletion action
		if (!confirm('Yakin ingin menghapus tiket ini?')) return;
		try {
			await axios.delete(
				`https://directus.eltamaprimaindo.com/items/TicketForm/${selectedTicket.rawId}`,
				{
					headers: { Authorization: `Bearer JaXaSE93k24zq7T2-vZyu3lgNOUgP8fz` }
				}
			);
			alert('Ticket berhasil dihapus');
			closeDetailModalTicket();
			fetchTickets();
		} catch (e) {
			alert('Gagal menghapus tiket');
		}
	}

	// Initialize auth, role, and department subscriptions on component mount
	onMount(() => {
		unsubAuth = isAuthenticated.subscribe((auth) => {
			// Check authentication status and redirect if not authenticated
			if (auth === undefined) return;
			authReady = true;
			isLoggedIn = !!auth;
			if (!auth) goto('/login');
		});
		unsubRole = userRole.subscribe((role) => {
			// Check user role and fetch data if admin
			if (role === undefined) return;
			roleReady = true;
			currentRole = role;
			if (role !== 'admin') goto('/login');
			// fetch data hanya jika sudah siap dan role benar
			if (authReady && roleReady && isLoggedIn && currentRole === 'admin') {
				fetchTickets();
				fetchFeedback();
				fetchEmployees().then((data) => {
					employees = data || [];
				});
			}
		});
		// Subscribe to user department changes
		unsubDept = userDepartment.subscribe((dept) => {
			adminDepartment = dept;
		});
	});

	// Clean up subscriptions when component is destroyed
	onDestroy(() => {
		if (unsubAuth) unsubAuth();
		if (unsubRole) unsubRole();
		if (unsubDept) unsubDept();
	});

	// Reactive statement: Filter tickets by admin's department
	$: filteredTickets = tickets.filter(
		(t) => t.department?.trim().toLowerCase() === adminDepartment?.trim().toLowerCase()
	);

	// Reactive statement: Log admin department changes for debugging
	$: if (adminDepartment !== undefined) {
		console.log('adminDepartment:', adminDepartment);
	}

	// Reactive: Cari data karyawan yang cocok dengan email admin
	$: myEmployee = employees.find(
		(e) =>
			e.email_company && e.email_company.trim().toLowerCase() === $userEmail?.trim().toLowerCase()
	);
</script>

<svelte:head>
	<title>Dashboard Admin</title>
	<meta
		name="description"
		content="Dashboard untuk Admin Helpdesk dengan statistik dan daftar tiket terbaru."
	/>
</svelte:head>
<!-- Check if authentication and role are ready -->
{#if !authReady || !roleReady}
	<div class="flex items-center justify-center min-h-screen text-lg text-blue-700">Loading...</div>
{:else}
	<div
		class="p-6 bg-gradient-to-br from-blue-100 via-blue-50 to-white min-h-screen animate-fade-in"
	>
		<h1 class="text-2xl font-bold text-blue-800 drop-shadow">Dashboard Admin</h1>
		<div class="flex justify-between items-center mb-6"></div>
		<!-- Stats -->
		<TicketStats tickets={filteredTickets} {feedbacks} />

		<!-- Quick Actions -->
		<div class="flex items-center gap-4 mb-6">
			<!-- Tombol Tiket Baru -->
			<button
				on:click={openTicketModal}
				class="bg-green-500 hover:bg-green-600 text-white px-6 py-3 rounded-xl text-center shadow-lg font-semibold transition transform hover:scale-105 animate-fade-in-up"
				>Tiket Baru</button
			>
		</div>

		<!-- Tickets Table -->
		<div class="bg-white p-4 rounded-xl shadow-lg mb-6 animate-fade-in-up">
			<h2 class="text-xl font-semibold mb-4 ml-2 text-blue-700 font-mono">List Tiket</h2>
			<TicketList
				tickets={filteredTickets}
				isAdmin={true}
				on:deleted={fetchTickets}
				on:openDetail={handleOpenDetailTicket}
				on:ticketUpdated={handleTicketUpdated}
			/>

			<!-- Feedback Table -->
			<h2 class="text-xl font-semibold mb-4 ml-2 text-blue-700 font-mono mt-4">List Feedback</h2>
			<FeedbackList
				{feedbacks}
				on:deleted={fetchFeedback}
				statusEditable={true}
				on:openImageFeedback={handleOpenImageFeedback}
				on:openDetailFeedback={handleOpenDetailFeedback}
			/>
		</div>
	</div>
{/if}

<!-- Modal for ticket image display -->
{#if showImageModalTicket}
	<div class="fixed inset-0 z-[100] flex items-center justify-center bg-black bg-opacity-40">
		<div
			class="bg-white rounded-xl shadow-2xl p-6 max-w-5xl w-full relative flex flex-col items-center animate-fade-in-up h-[70vh]"
		>
			<button
				class="absolute top-2 right-4 text-2xl font-bold text-gray-600 hover:text-red-600"
				on:click={closeImageTicket}>&times;</button
			>
			<h3 class="text-lg font-bold mb-4 text-blue-700">Lampiran Tiket</h3>
			<img src={imageUrlTicket} alt="Lampiran" class="max-h-[60vh] max-w-full rounded border" />
		</div>
	</div>
{/if}

<!-- Modal for feedback detail text display -->
{#if showDetailModalFeedback}
	<div class="fixed inset-0 z-[110] flex items-center justify-center bg-black bg-opacity-40">
		<div
			class="bg-white rounded-xl shadow-2xl p-6 max-w-4xl w-full relative flex flex-col items-start animate-fade-in-up"
			style="max-height: 80vh; overflow-y-auto"
		>
			<button
				class="absolute top-2 right-4 text-2xl font-bold text-gray-600 hover:text-red-600"
				on:click={closeDetailModalFeedback}>&times;</button
			>
			<h2 class="text-xl font-bold mb-4 text-blue-700 self-center">Detail Feedback</h2>
			
			<!-- Grid layout untuk feedback dan URL -->
			<div class="w-full grid grid-cols-1 md:grid-cols-2 gap-2 mb-2">
				<!-- Feedback column with vertical scrolling -->
				<div class="bg-blue-100 rounded-lg p-2">
					<h4 class="font-semibold text-blue-700 mb-1">Feedback</h4>
					<div class="max-h-[200px] overflow-y-auto pr-1 custom-scrollbar">
						<p class="text-gray-800 whitespace-pre-line leading-relaxed">
							{detailTextFeedback.split('\n\nURL:')[0].replace('Feedback:\n', '')}
						</p>
					</div>
				</div>
				
				<!-- URL column with vertical scrolling -->
				<div class="bg-blue-50 rounded-lg p-2">
					<h4 class="font-semibold text-blue-700 mb-1">URL</h4>
					<div class="max-h-[200px] overflow-y-auto pr-1 custom-scrollbar">
						<p class="text-gray-800 whitespace-pre-line leading-relaxed break-all">
							{detailTextFeedback.includes('URL:') ? detailTextFeedback.split('URL:\n')[1] : 'Tidak ada URL'}
						</p>
					</div>
				</div>
			</div>
			
			<!-- Lampiran section -->
			<div class="w-full flex flex-col items-center mt-2 border-t pt-2">
				<h3 class="text-lg font-semibold mb-2 text-blue-700 mt-2">Lampiran</h3>
				{#if imageUrlFeedback}
					<div class="p-2 bg-gray-50 rounded-lg border border-gray-100 shadow-sm">
						<img src={imageUrlFeedback} alt="Lampiran" class="max-h-[40vh] max-w-full rounded" />
					</div>
				{:else}
					<p class="text-gray-500 italic py-4">Lampiran tidak tersedia</p>
				{/if}
			</div>
		</div>
	</div>
{/if}

<!-- Modal for ticket detail information display -->
{#if showDetailModalTicket && selectedTicket}
	<div class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-40">
		<div
			class="bg-gradient-to-br from-blue-50 via-white to-blue-100 border border-blue-200 rounded-2xl shadow-4xl p-8 max-w-6xl w-full h-[60vh] overflow-y-auto relative animate-fade-in-up"
		>
			<button
				class="absolute top-3 right-5 text-2xl text-gray-400 hover:text-red-500 transition"
				on:click={closeDetailModalTicket}>&times;</button
			>
			<h2 class="text-xl font-extrabold mb-6 text-blue-700 drop-shadow gap-5">
				Detail Tiket <span
					class="bg-blue-100 text-blue-700 px-3 py-1 rounded-lg text-base font-semibold ml-2"
					>{selectedTicket?.id}</span
				>
			</h2>
			<table class="w-full text-left mb-4 border-separate border-spacing-y-1">
				<tbody class="space-y-2">
					{#each detailFieldsTicket as [key, value], idx}
						<!-- Display alternating row colors -->
						<tr class={idx % 2 === 0 ? 'bg-white' : 'bg-blue-50'}>
							<td class="font-semibold pr-4 py-2 capitalize text-blue-900 w-1/3">
								{key.replace(/_/g, ' ')}
							</td>
							<td class="py-2 text-gray-700 break-words">
								<!-- Check if field is detail and show button -->
								{#if key.toLowerCase() === 'detail'}
									<button
										class="px-3 py-1 bg-blue-500 hover:bg-blue-600 text-white rounded shadow text-xs font-bold"
										type="button"
										on:click={() => openTicketDetailModal(value)}
									>
										Lihat Detail
									</button>
									<!-- Check if field is attachment and show file button -->
								{:else if key.toLowerCase() === 'lampiran'}
									<!-- Check if ticket has photo attachment -->
									{#if selectedTicket.photo_ticket}
										<button
											class="px-3 py-1 bg-blue-500 hover:bg-blue-600 text-white rounded shadow text-xs font-bold"
											type="button"
											on:click={() => {
												imageUrlTicket = `https://directus.eltamaprimaindo.com/assets/${selectedTicket.photo_ticket}`;
												showImageModalTicket = true;
											}}
										>
											Lihat File
										</button>
									{:else}
										<span class="text-gray-400 italic">Tidak Tersedia</span>
									{/if}
									<!-- Check if value is array and join with comma -->
								{:else if Array.isArray(value)}
									{value.join(', ')}
								{:else}
									{value}
								{/if}
							</td>
						</tr>
					{/each}
				</tbody>
			</table>
			<!-- Check if current user is admin to show delete button -->
			{#if currentRole === 'admin'}
				<div class="mt-6 bg-red-50 rounded-lg p-4 border border-red-100 flex justify-center">
					<button
						class="flex-1 bg-red-500 text-white py-2 rounded-md hover:bg-red-600 transition font-semibold shadow"
						on:click={deleteTicket}
					>
						Hapus Tiket
					</button>
				</div>
			{/if}
		</div>
	</div>
{/if}

<!-- Modal for ticket detail text information -->
{#if showTicketDetailModal}
	<div class="fixed inset-0 z-[120] flex items-center justify-center bg-black bg-opacity-70">
		<div
			class="bg-white rounded-xl shadow-2xl p-6 max-w-5xl w-full relative flex flex-col items-center animate-fade-in-up h-[60vh]"
			style="max-height: 80vh; overflow-y: auto"
		>
			<button
				class="absolute top-3 right-4 text-2xl font-bold text-gray-600 hover:text-red-600"
				on:click={closeTicketDetailModal}>&times;</button
			>
			<h2 class="text-xl font-bold mb-4 text-blue-700">Detail Informasi Tiket</h2>
			<p class="text-gray-800 whitespace-pre-line text-justify">{ticketDetailText}</p>
		</div>
	</div>
{/if}

<!-- Modal untuk tiket form -->
{#if showTicketModal}
	<div
		class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-40 animate-fade-in"
	>
		<div class="bg-white rounded-lg shadow-xl p-0 max-w-2xl w-full relative animate-fade-in-up">
			<button
				class="absolute top-2 right-2 text-gray-500 hover:text-red-600 text-2xl font-bold z-10"
				on:click={closeTicketModal}>&times;</button
			>
			<TicketingForm
				onClose={closeTicketModal}
				on:submitted={handleTicketSubmitted}
				employee={myEmployee}
			/>
		</div>
	</div>
{/if}

<style>
	@keyframes fade-in {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}
	.animate-fade-in {
		animation: fade-in 0.7s cubic-bezier(0.4, 0, 0.2, 1) both;
	}
	@keyframes fade-in-up {
		from {
			opacity: 0;
			transform: translateY(30px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}
	.animate-fade-in-up {
		animation: fade-in-up 0.7s cubic-bezier(0.4, 0, 0.2, 1) both;
	}
	
	/* Custom scrollbar styling */
	.custom-scrollbar::-webkit-scrollbar {
		width: 6px;
	}
	.custom-scrollbar::-webkit-scrollbar-track {
		background: rgba(241, 245, 249, 0.5);
		border-radius: 10px;
	}
	.custom-scrollbar::-webkit-scrollbar-thumb {
		background: rgba(96, 165, 250, 0.5);
		border-radius: 10px;
	}
	.custom-scrollbar::-webkit-scrollbar-thumb:hover {
		background: rgba(59, 130, 246, 0.7);
	}
</style>
