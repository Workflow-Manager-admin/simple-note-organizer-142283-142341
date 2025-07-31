<script lang="ts">
	import { onMount } from 'svelte';
	import { notes, type Note } from '$lib/stores/notes';

	let noteList: Note[] = [];
	let selectedId: string | null = null;
	let currentNote: Note | null = null;
	let isEditing = false;
	let search = '';

	const unsubscribe = notes.subscribe(val => {
		noteList = val;
		if (selectedId) {
			const found = noteList.find(n => n.id === selectedId);
			if (!found && noteList.length > 0) selectedId = noteList[0].id;
			currentNote = noteList.find(n => n.id === selectedId) ?? null;
		} else {
			currentNote = noteList.length > 0 ? noteList[0] : null;
			selectedId = currentNote?.id ?? null;
		}
	});

	onMount(() => {
		if (noteList.length && !selectedId) {
			selectedId = noteList[0].id;
			currentNote = noteList[0];
		}
		return unsubscribe;
	});

	function handleNoteSelect(id: string) {
		selectedId = id;
		currentNote = noteList.find(n => n.id === id) ?? null;
		isEditing = false;
	}

	function handleNewNote() {
		const id = notes.addNote();
		selectedId = id;
		isEditing = true;
		// Wait for DOM to update so textarea becomes focused
		setTimeout(() => {
			const el = document.querySelector('textarea');
			if (el) (el as HTMLTextAreaElement).focus();
		}, 15);
	}

	function handleDeleteNote(id: string) {
		if (!confirm('Delete this note?')) return;
		notes.deleteNote(id);
		if (selectedId === id) {
			selectedId = noteList.length
				? noteList.find(n => n.id !== id)?.id ?? null
				: null;
			isEditing = false;
		}
	}

	function handleEdit() {
		isEditing = true;
		setTimeout(() => {
			const el = document.querySelector('textarea');
			if (el) (el as HTMLTextAreaElement).focus();
		}, 15);
	}

	let editTitle = '';
	let editContent = '';

	$: if (isEditing && currentNote) {
		editTitle = currentNote.title;
		editContent = currentNote.content;
	}
	function handleSaveEdit() {
		if (selectedId && currentNote) {
			notes.updateNote(selectedId, editTitle, editContent);
			isEditing = false;
		}
	}

	function handleCancelEdit() {
		isEditing = false;
	}

	// Filtered notes by search
	$: filteredNotes = search.length
		? noteList.filter(
				n =>
					n.title.toLowerCase().includes(search.toLowerCase()) ||
					n.content.toLowerCase().includes(search.toLowerCase())
		  )
		: noteList;
</script>

<svelte:head>
	<title>Notes</title>
</svelte:head>

<div class="notes-app">
	<aside class="sidebar">
		<div class="sidebar-header">
			<span class="app-title">My Notes</span>
			<button class="new-btn" title="New note" on:click={handleNewNote}>＋</button>
		</div>
		<input
			class="search"
			type="text"
			placeholder="Search notes"
			bind:value={search}
			autocomplete="off"
		/>
		<ul class="note-list">
			{#each filteredNotes as note (note.id)}
				<li class:selected={selectedId === note.id} title={note.title}>
					<button
						class="note-select-btn"
						type="button"
						tabindex="0"
						on:click={() => handleNoteSelect(note.id)}
						on:keydown={(e) => { if (e.key === 'Enter' || e.key === ' ') handleNoteSelect(note.id); }}
						aria-current={selectedId === note.id}
					>
						<span class="dot"></span>
						<span class="title">{note.title || 'Untitled'}</span>
					</button>
					<button class="delete-btn" on:click|stopPropagation={() => handleDeleteNote(note.id)} title="Delete note">✕</button>
				</li>
			{/each}
			{#if filteredNotes.length === 0}
				<li class="empty">No notes found.</li>
			{/if}
		</ul>
	</aside>
	<section class="main-area">
		<header class="note-header">
			{#if currentNote}
				{#if !isEditing}
					<h2>{currentNote.title || "Untitled"}</h2>
					<div class="actions">
						<button class="secondary-btn" on:click={handleEdit}>Edit</button>
					</div>
				{:else}
					<input class="edit-title" type="text" bind:value={editTitle} maxlength="64" />
					<div class="actions">
						<button class="primary-btn" on:click={handleSaveEdit}>Save</button>
						<button class="secondary-btn" on:click={handleCancelEdit}>Cancel</button>
					</div>
				{/if}
			{:else}
				<h2>No note selected</h2>
			{/if}
		</header>
		<div class="note-content">
			{#if currentNote}
				{#if isEditing}
					<textarea
						class="edit-content"
						bind:value={editContent}
						rows="12"
						placeholder="Write something…"
						maxlength="8192"
					></textarea>
				{:else}
					<div class="note-markdown">
						{#if currentNote.content}
							{currentNote.content}
						{:else}
							<i>No content</i>
						{/if}
					</div>
				{/if}
			{:else}
				<p>Select or create a note.</p>
			{/if}
		</div>
	</section>
</div>

<style>
.notes-app {
	display: flex;
	min-height: 65vh;
	box-shadow: 0 0 0.5rem 0 rgba(0,0,0,.06);
	border-radius: 15px;
	overflow: hidden;
	background: var(--color-bg-2, #fff);
	margin: 2rem auto;
	max-width: 900px;
}

.sidebar {
	width: 250px;
	display: flex;
	flex-direction: column;
	border-right: 1.5px solid #eee;
	background: var(--color-bg-1, #f8f9fa);
	padding: 0;
}

.sidebar-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 1rem;
	background: #fff;
	border-bottom: 1px solid #ececec;
}

.app-title {
	font-size: 1.25rem;
	font-weight: 600;
	color: var(--color-theme-1, #4F46E5);
}

.new-btn {
	background: var(--color-theme-2, #22D3EE);
	color: #fff;
	padding: .3em .7em;
	border: 0;
	border-radius: 6px;
	font-size: 1.1rem;
	cursor: pointer;
	transition: background .13s;
	font-weight: 700;
}
.new-btn:hover {
	background: var(--color-accent, #F59E42);
}

.search {
	margin: 1rem 1rem 0.5rem 1rem;
	padding: 0.55em 0.9em;
	border-radius: 7px;
	border: 1px solid #e2e8f0;
	font-size: 1em;
}

.note-list {
	list-style: none;
	margin: 0.5rem 0;
	padding: 0 0.5rem 0 0.5rem;
	flex: 1;
	overflow-y: auto;
}
.note-list li {
	display: flex;
	align-items: center;
	gap: 0.5rem;
	cursor: pointer;
	padding: 0.4rem 0.5rem 0.4rem 0.6rem;
	border-radius: 7px;
	transition: background 0.13s;
	font-size: 1em;
	position: relative;
}
.note-list li.selected {
	background: var(--color-theme-2, #22D3EE);
	font-weight: bold;
	color: white;
}
.note-list .note-select-btn {
	background: none;
	border: none;
	padding: 0;
	display: flex;
	align-items: center;
	flex: 1;
	text-align: left;
	cursor: pointer;
	font-size: inherit;
	color: inherit;
}
.note-list .dot {
	height: 9px;
	width: 9px;
	background: var(--color-accent, #F59E42);
	border-radius: 50%;
	display: inline-block;
	margin-right: 2px;
}
.note-list .title {
	flex: 1 1 auto;
	overflow: hidden;
	white-space: nowrap;
	text-overflow: ellipsis;
}
.note-list .delete-btn {
	border: none;
	background: none;
	color: #c54a4a;
	font-size: 1rem;
	cursor: pointer;
	padding: 0 0.2em;
	border-radius: 3px;
	transition: background 0.1s;
}
.note-list .delete-btn:hover {
	background: #fbeaec;
}
.note-list .empty {
	color: #999;
	text-align: center;
	font-size: 0.95em;
}

.main-area {
	flex: 1;
	min-width: 0;
	display: flex;
	flex-direction: column;
	background: var(--color-bg-2,#fff);
}
.note-header {
	display: flex;
	align-items: center;
	gap: 1rem;
	padding: 1.2rem 1.5rem 0.7rem 1.5rem;
	border-bottom: 1px solid #eee;
	background: #fff;
}
.note-header h2 {
	flex: 1;
	font-size: 1.1rem;
	font-weight: 500;
	margin: 0;
	color: var(--color-theme-1, #4F46E5);
}
.note-header .edit-title {
	flex: 1;
	font-size: 1.05rem;
	padding: 0.4em 0.7em;
	border: 1px solid #ddd;
	border-radius: 7px;
}
.actions {
	display: flex;
	gap: 0.6rem;
}
.primary-btn {
	background: var(--color-theme-1, #4F46E5);
	color: #fff;
	padding: 0.42em 1.1em;
	border-radius: 4px;
	border: none;
	font-weight: 500;
	cursor: pointer;
	transition: background 0.15s;
}
.primary-btn:hover {
	background: var(--color-accent, #F59E42);
}
.secondary-btn {
	background: var(--color-theme-2, #22D3EE);
	color: #fff;
	padding: 0.42em 1.1em;
	border-radius: 4px;
	border: none;
	font-weight: 500;
	cursor: pointer;
	transition: background 0.13s;
}
.secondary-btn:hover {
	background: var(--color-accent, #F59E42);
}
.note-content {
	padding: 1.5rem;
	flex: 1;
}
.note-content textarea.edit-content {
	width: 100%;
	height: 300px;
	resize: vertical;
	font-size: 1rem;
	font-family: var(--font-mono, monospace);
	border: 1px solid #cad5e0;
	padding: 0.7em;
	border-radius: 8px;
	color: #333;
}

.note-markdown {
	white-space: pre-wrap;
	font-family: inherit;
	min-height: 120px;
	font-size: 1.04em;
	color: #303233;
	border-radius: 6px;
	padding: 1em;
	background: #fbfbfb;
}

@media (max-width: 700px) {
	.notes-app {
		flex-direction: column;
		max-width: 98vw;
		box-shadow: none;
	}
	.sidebar {
		width: 100vw;
		border-right: none;
		border-bottom: 1.5px solid #ececec;
	}
	.main-area {
		padding: 0 0.1rem;
	}
	.note-header, .note-content { padding-left: 5vw; padding-right: 5vw; }
}
</style>
