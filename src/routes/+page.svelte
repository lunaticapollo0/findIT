<script>
  import { stopPropagation, createBubbler } from 'svelte/legacy';

  const bubble = createBubbler();
  import { onMount } from 'svelte';

  let user = $state(null);

  function mockMicrosoftSignIn() {
    // mock login. final product will be shipped with microsoft student account login
    user = { name: 'BN Nagaraajan', email: 'bl.en.u4ece26207@bl.students.amrita.edu', initials: 'BN' };
  }

  function signOut() {
    user = null;
  }

  // ---------- helpers ----------
  function hoursAgo(h) { return Date.now() - h * 60 * 60 * 1000; }
  function daysAgo(d) { return Date.now() - d * 24 * 60 * 60 * 1000; }

  function formatDate(ts) {
    if (!ts) return '';
    const d = new Date(ts);
    const day = d.getDate();
    const month = d.toLocaleString('en-US', { month: 'long' });
    const year = d.getFullYear();
    let hrs = d.getHours();
    const mins = d.getMinutes().toString().padStart(2, '0');
    const ampm = hrs >= 12 ? 'PM' : 'AM';
    hrs = hrs % 12;
    hrs = hrs ? hrs : 12;
    return `${day} ${month} ${year} ${hrs}:${mins} ${ampm}`;
  }

  function statusLabelFor(item) {
    if (item.status === 'Claimed') return `Claimed on ${formatDate(item.resolvedAt || item.postedAt)}`;
    if (item.status === 'Returned') return `Returned on ${formatDate(item.resolvedAt || item.postedAt)}`;
    if (item.status === 'Lost') return `Lost on ${formatDate(item.postedAt)}`;
    return `Found on ${formatDate(item.postedAt)}`;
  }

  function mapsUrl(location) {
    return `https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(location + ', Amrita Bengaluru')}`;
  }

  function toDatetimeLocal(ts) {
    const d = new Date(ts);
    const pad = n => n.toString().padStart(2, '0');
    return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}T${pad(d.getHours())}:${pad(d.getMinutes())}`;
  }

  function isThisMonth(ts) {
    if (!ts) return false;
    const d = new Date(ts);
    const now = new Date();
    return d.getFullYear() === now.getFullYear() && d.getMonth() === now.getMonth();
  }

  // ---------- fake data ----------
  let items = $state([
    {
      id: 1,
      title: 'Black wired earphones',
      status: 'Lost',
      category: 'Electronics',
      location: 'boys hostel canteen',
      description: 'Standard black wired earphones with a small tangle in the cable near the jack. Left one behind on a canteen table around lunchtime.',
      phone: '9876543210',
      images: [{ label: 'Front', src: null }],
      postedAt: hoursAgo(2),
      resolvedAt: null,
      comments: [],
      reports: [],
      postedBy: 'bl.en.u4ece26207@bl.students.amrita.edu'
    },
    {
      id: 2,
      title: 'College ID card - CSE',
      status: 'Found',
      category: 'ID cards',
      location: 'Library reading hall',
      description: 'Found tucked between books on the second floor reading hall. Handed over to the reading hall desk if unclaimed by evening.',
      phone: '9876500001',
      images: [{ label: 'Front', src: null }],
      postedAt: hoursAgo(5),
      resolvedAt: null,
      comments: [],
      reports: [],
      postedBy: 'someone.else@bl.students.amrita.edu'
    },
    {
      id: 3,
      title: 'Steel water bottle, blue',
      status: 'Found',
      category: 'Accessories',
      location: 'Block E classroom E205',
      description: 'Blue steel bottle with a dented cap, found under a bench after the last class of the day.',
      phone: '9876500002',
      images: [{ label: 'Front', src: null }],
      postedAt: daysAgo(1),
      resolvedAt: null,
      comments: [],
      reports: [],
      postedBy: 'someone.else@bl.students.amrita.edu'
    },
    {
      id: 4,
      title: 'Brown leather wallet',
      status: 'Claimed',
      category: 'Accessories',
      location: 'Amriteshwari hall',
      description: 'Brown leather wallet with a couple of cards inside, no cash. Claimed by the owner shortly after posting.',
      phone: '9876500003',
      images: [{ label: 'Front', src: null }],
      postedAt: daysAgo(1),
      resolvedAt: hoursAgo(20),
      comments: [],
      reports: [],
      postedBy: 'someone.else@bl.students.amrita.edu'
    },
    {
      id: 5,
      title: 'Smartwatch, black strap',
      status: 'Returned',
      category: 'Electronics',
      location: 'Girls Gym entrance',
      description: 'Black strap smartwatch, screen slightly scratched. Returned to the owner directly.',
      phone: '9876500004',
      images: [{ label: 'Front', src: null }],
      postedAt: daysAgo(2),
      resolvedAt: daysAgo(1),
      comments: [],
      reports: [],
      postedBy: 'someone.else@bl.students.amrita.edu'
    }
  ]);

  const categories = ['Electronics', 'ID cards', 'Books/notes', 'Accessories'];
  const icons = {
    Electronics: '🎧',
    'ID cards': '🪪',
    'Books/notes': '📓',
    Accessories: '🎒'
  };
  const imageLabels = ['Front', 'Back', 'Close-up', 'Other'];
  const reportReasons = ['Spam', 'Inappropriate content', 'Fake or misleading', 'Already resolved', 'Other'];

  // ---------- browse state ----------
  let view = $state('active'); // 'active' | 'archives' | 'mine'
  let typeFilter = $state('All items'); // 'All items' | 'Lost' | 'Found'
  let categoryFilters = $state(new Set());
  let search = $state('');
  let sortOrder = $state('newest'); // 'newest' | 'oldest'

  function toggleCategory(cat) {
    if (categoryFilters.has(cat)) categoryFilters.delete(cat);
    else categoryFilters.add(cat);
    categoryFilters = categoryFilters;
  }

  let reunitedCount = $derived(items.filter(
    i => (i.status === 'Returned') && isThisMonth(i.resolvedAt)
  ).length);

  let filteredItems = $derived(items
    .filter(item => {
      if (view === 'mine') return !!user && item.postedBy === user.email;
      const isArchived = item.status === 'Claimed' || item.status === 'Returned';
      if (view === 'archives' && !isArchived) return false;
      if (view === 'active' && isArchived) return false;
      if (view === 'active' && typeFilter !== 'All items' && item.status !== typeFilter) return false;
      return true;
    })
    .filter(item => categoryFilters.size === 0 || categoryFilters.has(item.category))
    .filter(item =>
      !search ||
      item.title.toLowerCase().includes(search.toLowerCase()) ||
      item.location.toLowerCase().includes(search.toLowerCase())
    )
    .sort((a, b) => (sortOrder === 'newest' ? b.postedAt - a.postedAt : a.postedAt - b.postedAt)));

  // ---------- post form state ----------
  let showPostForm = $state(false);
  let newTitle = $state('');
  let newStatus = $state('Lost');
  let newCategory = $state('Electronics');
  let newLocation = $state('');
  let newDescription = $state('');
  let newPhone = $state('');
  let newDate = $state(toDatetimeLocal(Date.now()));
  let newImages = $state([]); // [{ preview, label }]
  let formError = $state('');

  function nextDefaultLabel() {
    return imageLabels[newImages.length] || 'Other';
  }

  function handleImagesChange(e) {
    const files = Array.from(e.target.files || []);
    if (!files.length) return;

    formError = '';
    for (const file of files) {
      if (newImages.length >= 4) {
        formError = 'You can add up to 4 photos.';
        break;
      }
      if (!file.type.startsWith('image/')) {
        formError = 'Please choose image files only.';
        continue;
      }
      if (file.size > 5 * 1024 * 1024) {
        formError = 'Each image must be under 5MB.';
        continue;
      }
      const label = nextDefaultLabel();
      const reader = new FileReader();
      reader.onload = (ev) => {
        newImages = [...newImages, { preview: ev.target.result, label }];
      };
      reader.readAsDataURL(file);
    }
    e.target.value = '';
  }

  function removeNewImage(idx) {
    newImages = newImages.filter((_, i) => i !== idx);
  }

  function isValidPhone(phone) {
    const digits = phone.replace(/[^0-9]/g, '');
    return digits.length >= 7;
  }

  function addItem() {
    formError = '';

    if (!newTitle || !newLocation) {
      formError = 'Please fill in the title and location.';
      return;
    }
    if (!newPhone || !isValidPhone(newPhone)) {
      formError = 'A valid contact phone number is required.';
      return;
    }
    if (!newDate) {
      formError = 'Please pick a date and time.';
      return;
    }

    items = [
      {
        id: Date.now(),
        title: newTitle,
        status: newStatus,
        category: newCategory,
        location: newLocation,
        description: newDescription,
        phone: newPhone,
        images: newImages.length ? newImages.map(img => ({ label: img.label, src: img.preview })) : [{ label: 'Front', src: null }],
        postedAt: new Date(newDate).getTime(),
        resolvedAt: null,
        comments: [],
        reports: [],
        postedBy: user ? user.email : null
      },
      ...items
    ];

    resetForm();
    showPostForm = false;
  }

  function resetForm() {
    newTitle = '';
    newStatus = 'Lost';
    newCategory = 'Electronics';
    newLocation = '';
    newDescription = '';
    newPhone = '';
    newDate = toDatetimeLocal(Date.now());
    newImages = [];
    formError = '';
  }

  function closeForm() {
    showPostForm = false;
    resetForm();
  }

  // ---------- item details state ----------
  let selectedItem = $state(null);
  let activeImageIdx = $state(0);
  let commentName = $state('');
  let commentText = $state('');

  function openDetail(item) {
    selectedItem = item;
    activeImageIdx = 0;
    commentName = '';
    commentText = '';
  }

  function closeDetail() {
    selectedItem = null;
  }

  function setActiveImage(idx) {
    activeImageIdx = idx;
  }

  function changeStatus(newVal) {
    selectedItem.status = newVal;
    if (newVal === 'Claimed' || newVal === 'Returned') {
      selectedItem.resolvedAt = Date.now();
    } else {
      selectedItem.resolvedAt = null;
    }
    items = items;
  }

  function addComment() {
    if (!commentText.trim()) return;
    const comment = {
      id: Date.now(),
      name: commentName.trim() || 'Anonymous',
      text: commentText.trim(),
      at: Date.now()
    };
    selectedItem.comments = [...selectedItem.comments, comment];
    items = items;
    commentText = '';
  }

  // ---------- report state ----------
  let reportingItem = $state(null);
  let reportReason = $state(reportReasons[0]);
  let reportDetails = $state('');

  function openReport(item) {
    reportingItem = item;
    reportReason = reportReasons[0];
    reportDetails = '';
  }

  function closeReport() {
    reportingItem = null;
  }

  function submitReport() {
    if (!reportingItem) return;
    const report = {
      id: Date.now(),
      reason: reportReason,
      details: reportDetails.trim(),
      at: Date.now()
    };
    reportingItem.reports = [...(reportingItem.reports || []), report];
    items = items;
    closeReport();
    showToast("Report submitted — thanks for flagging this.");
  }

  // ---------- share / copy link ----------
  let toastMessage = $state('');
  let toastTimer;

  function showToast(msg) {
    toastMessage = msg;
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => { toastMessage = ''; }, 2500);
  }

  function buildShareUrl(item) {
    const base = `${window.location.origin}${window.location.pathname}`;
    return `${base}?item=${item.id}`;
  }

  async function shareItem(item) {
    const url = buildShareUrl(item);
    if (navigator.share) {
      try {
        await navigator.share({ title: item.title, text: 'Check this out on FindIt On Campus', url });
        return;
      } catch (e) {
        // user cancelled share sheet, or share failed — fall through to clipboard copy
      }
    }
    try {
      await navigator.clipboard.writeText(url);
      showToast('Link copied to clipboard!');
    } catch (e) {
      showToast('Could not copy link.');
    }
  }

  // open a shared item link directly, once signed in
  onMount(() => {
    const params = new URLSearchParams(window.location.search);
    const itemId = params.get('item');
    if (itemId) {
      const found = items.find(i => i.id === Number(itemId));
      if (found) selectedItem = found;
    }
  });
</script>

<main>
  {#if !user}
    <div class="login-screen">
      <span class="pin big">📍</span>
      <h2>FindIt On Campus</h2>
      <p class="sub-text">Amrita Bengaluru · Lost &amp; Found</p>
      <button class="ms-btn" onclick={mockMicrosoftSignIn}>
        <span class="ms-logo">
          <span style="background:#f25022"></span><span style="background:#7fba00"></span><span style="background:#00a4ef"></span><span style="background:#ffb900"></span>
        </span>
        Sign in with Microsoft
      </button>
      <p class="hint">Use your @bl.students.amrita.edu account</p>
    </div>
  {:else}
  <header>
    <div class="brand">
      <span class="pin">📍</span>
      <span class="name">FindIt On Campus</span>
      <span class="sub">Amrita Bengaluru</span>
    </div>
    <div class="header-right">
      <input class="search" type="text" placeholder="Search item, location..." bind:value={search} />
      <button class="post-btn" onclick={() => showPostForm = true}>+ Post item</button>
      <div class="avatar" onclick={signOut} title="Click to sign out">{user.initials}</div>
    </div>
  </header>

  <div class="stats-banner">
    <span class="stats-icon">🎉</span>
    <span><strong>{reunitedCount}</strong> item{reunitedCount === 1 ? '' : 's'} reunited with {reunitedCount === 1 ? 'its' : 'their'} owner{reunitedCount === 1 ? '' : 's'} this month</span>
  </div>

  <div class="body">
    <aside>
      <p class="label">Browse</p>
      <button class="filter-btn" class:active={view === 'active'} onclick={() => { view = 'active'; }}>🗂️ Active items</button>
      <button class="filter-btn" class:active={view === 'mine'} onclick={() => { view = 'mine'; }}>🙋 My posts</button>
      <button class="filter-btn" class:active={view === 'archives'} onclick={() => { view = 'archives'; }}>📦 Archives</button>

      {#if view === 'active'}
        <p class="label" style="margin-top:20px;">Filter by</p>
        <button class="filter-btn" class:active={typeFilter === 'All items'} onclick={() => typeFilter = 'All items'}>All items</button>
        <button class="filter-btn" class:active={typeFilter === 'Lost'} onclick={() => typeFilter = 'Lost'}>Lost</button>
        <button class="filter-btn" class:active={typeFilter === 'Found'} onclick={() => typeFilter = 'Found'}>Found</button>
      {/if}

      <p class="label" style="margin-top:20px;">Category</p>
      {#each categories as cat}
        <label class="checkbox-row">
          <input type="checkbox" checked={categoryFilters.has(cat)} onchange={() => toggleCategory(cat)} />
          {cat}
        </label>
      {/each}
    </aside>

    <section class="grid-wrap">
      <div class="section-header">
        <p class="section-title">
          {view === 'active' ? 'Active items' : view === 'archives' ? 'Archives' : 'My posts'}
          <span class="count">({filteredItems.length})</span>
        </p>
        <select class="sort-select" bind:value={sortOrder}>
          <option value="newest">Newest first</option>
          <option value="oldest">Oldest first</option>
        </select>
      </div>

      <div class="grid">
        {#if (view === 'archives' || view === 'mine') && filteredItems.length === 0}
          <p class="empty-note">{view === 'archives' ? 'No claimed or returned items yet.' : "You haven't posted anything yet."}</p>
        {/if}

        {#each filteredItems as item (item.id)}
          <div class="card" onclick={() => openDetail(item)}>
            <button class="report-flag" onclick={stopPropagation(() => openReport(item))} title="Report this post">🚩</button>
            <div class="thumb">
              {#if item.images[0] && item.images[0].src}
                <img src={item.images[0].src} alt={item.title} class="thumb-img" />
              {:else}
                {icons[item.category] ?? '📦'}
              {/if}
              {#if item.images.length > 1}
                <span class="img-count">📷 {item.images.length}</span>
              {/if}
            </div>
            <span class="badge {item.status.toLowerCase()}">{item.status}</span>
            <p class="title">{item.title}</p>
            <p class="meta">📍 {item.location}</p>
            <p class="meta">{statusLabelFor(item)}</p>
          </div>
        {/each}

        {#if view === 'active' || view === 'mine'}
          <button class="new-card" onclick={() => showPostForm = true}>
            <span class="plus">+</span>
            <span>Post a new item</span>
          </button>
        {/if}
      </div>
    </section>
  </div>

  {#if showPostForm}
    <div class="overlay" onclick={closeForm}>
      <div class="modal" onclick={stopPropagation(bubble('click'))}>
        <h3>Post an item</h3>

        <label>Title
          <input type="text" bind:value={newTitle} placeholder="e.g. Black wired earphones" />
        </label>
        <label>Type
          <select bind:value={newStatus}>
            <option>Lost</option>
            <option>Found</option>
          </select>
        </label>
        <label>Category
          <select bind:value={newCategory}>
            {#each categories as cat}
              <option>{cat}</option>
            {/each}
          </select>
        </label>
        <label>{newStatus === 'Lost' ? 'Last known location' : 'Found at'}
          <input
            type="text"
            bind:value={newLocation}
            placeholder={newStatus === 'Lost' ? 'e.g. Library' : 'e.g. ATM road'}
          />
        </label>
        <label>Date &amp; time
          <input type="datetime-local" bind:value={newDate} />
        </label>
        <label>Description
          <textarea rows="3" bind:value={newDescription} placeholder="Colour, brand, any identifying details..."></textarea>
        </label>
        <label>Contact phone number
          <input type="tel" bind:value={newPhone} placeholder="e.g. 9876543210" />
        </label>

        <label>Photos (up to 4 — front, back, close-up...)
          <input type="file" accept="image/*" multiple onchange={handleImagesChange} />
        </label>

        {#if newImages.length}
          <div class="image-list">
            {#each newImages as img, i}
              <div class="image-item">
                <img src={img.preview} alt="preview" />
                <select bind:value={img.label}>
                  {#each imageLabels as lbl}
                    <option>{lbl}</option>
                  {/each}
                </select>
                <button type="button" class="remove-img" onclick={() => removeNewImage(i)}>✕</button>
              </div>
            {/each}
          </div>
        {/if}

        {#if formError}
          <p class="form-error">{formError}</p>
        {/if}

        <div class="modal-actions">
          <button class="cancel" onclick={closeForm}>Cancel</button>
          <button class="submit" onclick={addItem}>Post</button>
        </div>
      </div>
    </div>
  {/if}

  {#if selectedItem}
    <div class="overlay" onclick={closeDetail}>
      <div class="detail-modal" onclick={stopPropagation(bubble('click'))}>
        <button class="detail-close" onclick={closeDetail}>✕</button>

        <div class="detail-grid">
          <div class="detail-images">
            <div class="detail-main-image">
              {#if selectedItem.images[activeImageIdx] && selectedItem.images[activeImageIdx].src}
                <img src={selectedItem.images[activeImageIdx].src} alt={selectedItem.title} />
              {:else}
                <span class="detail-icon">{icons[selectedItem.category] ?? '📦'}</span>
              {/if}
            </div>
            {#if selectedItem.images.length > 1}
              <div class="thumb-strip">
                {#each selectedItem.images as img, i}
                  <button
                    class="thumb-strip-item"
                    class:active={i === activeImageIdx}
                    onclick={() => setActiveImage(i)}
                  >
                    {#if img.src}
                      <img src={img.src} alt={img.label} />
                    {:else}
                      <span>{icons[selectedItem.category] ?? '📦'}</span>
                    {/if}
                    <span class="thumb-strip-label">{img.label}</span>
                  </button>
                {/each}
              </div>
            {/if}
          </div>

          <div class="detail-info">
            <span class="badge {selectedItem.status.toLowerCase()}">{selectedItem.status}</span>
            <h2 class="detail-title">{selectedItem.title}</h2>
            <p class="detail-date">{statusLabelFor(selectedItem)}</p>

            

            {#if selectedItem.description}
              <p class="detail-desc">{selectedItem.description}</p>
            {/if}

            <p class="detail-contact">📞 <a href="tel:{selectedItem.phone}">{selectedItem.phone}</a></p>

            <div class="action-row">
              <button class="share-btn" onclick={() => shareItem(selectedItem)}>🔗 Copy link</button>
              <button class="report-btn" onclick={() => openReport(selectedItem)}>🚩 Report this post</button>
            </div>

            <div class="status-update">
              <label>Update status
                <select value={selectedItem.status} onchange={(e) => changeStatus(e.target.value)}>
                  <option>Lost</option>
                  <option>Found</option>
                  <option>Claimed</option>
                  <option>Returned</option>
                </select>
              </label>
            </div>

            <div class="comments">
              <p class="label">Comments ({selectedItem.comments.length})</p>
              {#each selectedItem.comments as c (c.id)}
                <div class="comment">
                  <p class="comment-head"><strong>{c.name}</strong> <span>{formatDate(c.at)}</span></p>
                  <p class="comment-text">{c.text}</p>
                </div>
              {/each}

              <div class="comment-form">
                <input type="text" placeholder="Your name (optional)" bind:value={commentName} />
                <textarea rows="2" placeholder="Add a comment..." bind:value={commentText}></textarea>
                <button class="submit" onclick={addComment}>Post comment</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  {/if}

  {#if reportingItem}
    <div class="overlay" onclick={closeReport}>
      <div class="modal" onclick={stopPropagation(bubble('click'))}>
        <h3>Report post</h3>
        <p class="report-target">"{reportingItem.title}"</p>

        <label>Reason
          <select bind:value={reportReason}>
            {#each reportReasons as reason}
              <option>{reason}</option>
            {/each}
          </select>
        </label>
        <label>Additional details (optional)
          <textarea rows="3" bind:value={reportDetails} placeholder="Anything the moderators should know..."></textarea>
        </label>

        <div class="modal-actions">
          <button class="cancel" onclick={closeReport}>Cancel</button>
          <button class="submit" onclick={submitReport}>Submit report</button>
        </div>
      </div>
    </div>
  {/if}

  {#if toastMessage}
    <div class="toast">{toastMessage}</div>
  {/if}
  {/if}
</main>

<style>
  :global(body) {
    background: #16171a;
    margin: 0;
    font-family: system-ui, sans-serif;
    color: #eee;
  }

  main {
    max-width: 960px;
    margin: 20px auto;
    background: #1c1d21;
    border: 1px solid #2c2d31;
    border-radius: 12px;
    overflow: hidden;
    position: relative;
  }

  .login-screen {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 60px 20px;
    text-align: center;
  }
  .login-screen .big { font-size: 40px; margin-bottom: 10px; }
  .login-screen h2 { margin: 0 0 4px; }
  .sub-text { color: #888; font-size: 13px; margin: 0 0 28px; }

  .ms-btn {
    display: flex;
    align-items: center;
    gap: 12px;
    background: #fff;
    color: #1b1b1b;
    border: 1px solid #34353a;
    border-radius: 4px;
    height: 42px;
    padding: 0 16px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
  }
  .ms-logo {
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr 1fr;
    gap: 1px;
    width: 16px;
    height: 16px;
  }
  .ms-logo span { display: block; }

  .hint { color: #666; font-size: 12px; margin-top: 14px; }

  .avatar { cursor: pointer; }

  header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 20px;
    border-bottom: 1px solid #2c2d31;
  }

  .brand { display: flex; align-items: center; gap: 8px; }
  .name { font-weight: 600; }
  .sub { color: #888; font-size: 13px; }

  .header-right { display: flex; align-items: center; gap: 10px; }

  .search {
    width: 220px;
    height: 34px;
    background: #26272b;
    border: 1px solid #34353a;
    border-radius: 8px;
    color: #eee;
    padding: 0 10px;
  }

  .post-btn {
    background: #fff;
    color: #16171a;
    border: none;
    border-radius: 8px;
    height: 36px;
    padding: 0 14px;
    font-weight: 600;
    cursor: pointer;
  }

  .avatar {
    width: 32px; height: 32px;
    border-radius: 50%;
    background: #2d4f8a;
    color: #9cc0f5;
    display: flex; align-items: center; justify-content: center;
    font-size: 12px; font-weight: 600;
  }

  .stats-banner {
    display: flex;
    align-items: center;
    gap: 8px;
    margin: 14px 20px 0;
    padding: 10px 14px;
    background: #173821;
    border: 1px solid #1f4a2c;
    border-radius: 10px;
    font-size: 13px;
    color: #a9e6bb;
  }
  .stats-icon { font-size: 15px; }
  .stats-banner strong { color: #d7f7de; }

  .body { display: flex; gap: 16px; padding: 16px 20px; }

  aside { width: 170px; flex-shrink: 0; }
  .label { font-size: 12px; color: #888; margin: 0 0 8px; }

  .filter-btn {
    display: block;
    width: 100%;
    text-align: left;
    background: #26272b;
    border: 1px solid #34353a;
    color: #eee;
    border-radius: 8px;
    padding: 8px 12px;
    margin-bottom: 6px;
    cursor: pointer;
  }
  .filter-btn.active {
    background: #1e3a63;
    border-color: #2d5fa5;
    color: #9cc0f5;
  }

  .checkbox-row {
    display: flex; align-items: center; gap: 8px;
    font-size: 13px; color: #ccc; margin-bottom: 6px;
  }

  .grid-wrap {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 12px;
    min-width: 0;
  }

  .section-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .section-title {
    margin: 0;
    font-size: 13px;
    font-weight: 600;
    color: #ddd;
  }
  .section-title .count {
    font-weight: 400;
    color: #888;
    margin-left: 4px;
  }
  .sort-select {
    background: #26272b;
    border: 1px solid #34353a;
    color: #eee;
    border-radius: 8px;
    height: 30px;
    padding: 0 8px;
    font-size: 12px;
  }

  .grid {
    display: grid;
    grid-template-columns: repeat(3, minmax(0,1fr));
    gap: 12px;
    align-content: start;
  }

  .empty-note {
    grid-column: 1 / -1;
    color: #777;
    font-size: 13px;
    text-align: center;
    padding: 30px 0;
  }

  .card {
    position: relative;
    background: #26272b;
    border-radius: 12px;
    padding: 12px;
    cursor: pointer;
    transition: transform 0.1s ease, border-color 0.1s ease;
    border: 1px solid transparent;
  }
  .card:hover {
    border-color: #34353a;
    transform: translateY(-1px);
  }

  .report-flag {
    position: absolute;
    top: 8px;
    right: 8px;
    width: 24px;
    height: 24px;
    border-radius: 50%;
    background: rgba(0,0,0,0.55);
    border: 1px solid #3a3b40;
    color: #eee;
    font-size: 11px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    z-index: 1;
  }
  .report-flag:hover { background: rgba(0,0,0,0.75); }

  .thumb {
    position: relative;
    height: 90px;
    border-radius: 8px;
    background: #303136;
    display: flex; align-items: center; justify-content: center;
    font-size: 28px;
    margin-bottom: 8px;
    overflow: hidden;
  }

  .thumb-img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    border-radius: 8px;
  }

  .img-count {
    position: absolute;
    bottom: 4px;
    right: 4px;
    background: rgba(0,0,0,0.6);
    color: #eee;
    font-size: 10px;
    padding: 2px 6px;
    border-radius: 6px;
  }

  .badge {
    display: inline-block;
    font-size: 11px;
    padding: 2px 8px;
    border-radius: 6px;
  }
  .badge.lost { background: #4a1b1e; color: #f5a3a8; }
  .badge.found { background: #173821; color: #7fd99a; }
  .badge.claimed { background: #4a3a14; color: #f5cf7a; }
  .badge.returned { background: #17293a; color: #7ab8f5; }

  .title { font-size: 13px; font-weight: 600; margin: 6px 0 2px; }
  .meta { font-size: 12px; color: #999; margin: 0; }

  .new-card {
    background: transparent;
    border: 1px dashed #444;
    border-radius: 12px;
    color: #888;
    display: flex; flex-direction: column; align-items: center; justify-content: center;
    gap: 4px;
    cursor: pointer;
    min-height: 130px;
  }
  .new-card .plus { font-size: 22px; }

  .overlay {
    position: fixed; inset: 0;
    background: rgba(0,0,0,0.5);
    display: flex; align-items: center; justify-content: center;
    padding: 20px;
    box-sizing: border-box;
    z-index: 10;
  }

  .modal {
    background: #1c1d21;
    border: 1px solid #34353a;
    border-radius: 12px;
    padding: 20px;
    width: 320px;
    max-height: 90vh;
    overflow-y: auto;
  }
  .modal h3 { margin: 0 0 14px; }
  .report-target {
    font-size: 12px;
    color: #999;
    margin: -8px 0 14px;
    font-style: italic;
  }
  .modal label {
    display: block;
    font-size: 12px;
    color: #999;
    margin-bottom: 10px;
  }
  .modal input, .modal select, .modal textarea {
    display: block;
    width: 100%;
    margin-top: 4px;
    background: #26272b;
    border: 1px solid #34353a;
    border-radius: 8px;
    color: #eee;
    padding: 0 10px;
    box-sizing: border-box;
    font-family: inherit;
  }
  .modal input, .modal select { height: 34px; }
  .modal textarea { padding: 8px 10px; resize: vertical; }
  .modal input[type="file"] {
    height: auto;
    padding: 6px 0;
    background: transparent;
    border: none;
  }

  .image-list {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 12px;
  }
  .image-item {
    width: 92px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .image-item img {
    width: 92px;
    height: 70px;
    object-fit: cover;
    border-radius: 8px;
    display: block;
  }
  .image-item select {
    height: 26px;
    font-size: 11px;
    padding: 0 4px;
  }
  .remove-img {
    background: transparent;
    border: 1px solid #34353a;
    color: #ccc;
    border-radius: 6px;
    padding: 2px 0;
    font-size: 11px;
    cursor: pointer;
  }

  .form-error {
    color: #f5a3a8;
    font-size: 12px;
    margin: -4px 0 10px;
  }

  .modal-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 12px;
  }
  .cancel, .submit {
    border-radius: 8px;
    padding: 8px 14px;
    cursor: pointer;
    border: 1px solid #34353a;
    font-family: inherit;
  }
  .cancel { background: transparent; color: #ccc; }
  .submit { background: #fff; color: #16171a; font-weight: 600; border: none; }

  /* ---- item details modal ---- */
  .detail-modal {
    position: relative;
    background: #1c1d21;
    border: 1px solid #34353a;
    border-radius: 12px;
    padding: 24px;
    width: 100%;
    max-width: 760px;
    max-height: 90vh;
    overflow-y: auto;
  }

  .detail-close {
    position: absolute;
    top: 14px;
    right: 14px;
    background: #26272b;
    border: 1px solid #34353a;
    color: #ccc;
    border-radius: 50%;
    width: 28px;
    height: 28px;
    cursor: pointer;
  }

  .detail-grid {
    display: grid;
    grid-template-columns: 300px 1fr;
    gap: 24px;
  }

  .detail-main-image {
    height: 220px;
    background: #26272b;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }
  .detail-main-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
  .detail-icon { font-size: 56px; }

  .thumb-strip {
    display: flex;
    gap: 8px;
    margin-top: 8px;
    flex-wrap: wrap;
  }
  .thumb-strip-item {
    background: #26272b;
    border: 1px solid #34353a;
    border-radius: 8px;
    padding: 4px;
    width: 68px;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 3px;
  }
  .thumb-strip-item.active {
    border-color: #2d5fa5;
  }
  .thumb-strip-item img {
    width: 100%;
    height: 44px;
    object-fit: cover;
    border-radius: 5px;
  }
  .thumb-strip-item span:not(.thumb-strip-label) {
    font-size: 20px;
  }
  .thumb-strip-label {
    font-size: 10px;
    color: #999;
  }

  .detail-title { margin: 8px 0 2px; font-size: 20px; }
  .detail-date { color: #999; font-size: 13px; margin: 0 0 12px; }
  .detail-location {
    font-size: 13px;
    color: #ccc;
    margin: 0 0 10px;
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }
  .map-link {
    color: #9cc0f5;
    font-size: 12px;
    text-decoration: none;
    border: 1px solid #2d5fa5;
    border-radius: 6px;
    padding: 2px 8px;
  }
  .map-link:hover { background: #1e3a63; }

  .detail-desc {
    font-size: 13px;
    color: #ccc;
    line-height: 1.5;
    margin: 0 0 12px;
  }

  .detail-contact { font-size: 13px; margin: 0 0 16px; }
  .detail-contact a { color: #9cc0f5; text-decoration: none; }

  .action-row {
    display: flex;
    gap: 8px;
    margin-bottom: 18px;
    flex-wrap: wrap;
  }
  .share-btn, .report-btn {
    background: #26272b;
    border: 1px solid #34353a;
    color: #ccc;
    border-radius: 8px;
    padding: 7px 12px;
    font-size: 12px;
    cursor: pointer;
    font-family: inherit;
  }
  .share-btn:hover { border-color: #2d5fa5; color: #9cc0f5; }
  .report-btn:hover { border-color: #7a3030; color: #f5a3a8; }

  .status-update {
    margin-bottom: 20px;
  }
  .status-update label {
    font-size: 12px;
    color: #999;
    display: block;
  }
  .status-update select {
    margin-top: 4px;
    height: 34px;
    background: #26272b;
    border: 1px solid #34353a;
    border-radius: 8px;
    color: #eee;
    padding: 0 10px;
  }

  .comments {
    border-top: 1px solid #2c2d31;
    padding-top: 14px;
  }
  .comment {
    background: #232428;
    border-radius: 8px;
    padding: 8px 10px;
    margin-bottom: 8px;
  }
  .comment-head {
    margin: 0 0 2px;
    font-size: 12px;
    color: #ccc;
    display: flex;
    justify-content: space-between;
  }
  .comment-head span { color: #777; font-weight: 400; }
  .comment-text { margin: 0; font-size: 13px; color: #ddd; }

  .comment-form {
    margin-top: 10px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .comment-form input, .comment-form textarea {
    background: #26272b;
    border: 1px solid #34353a;
    border-radius: 8px;
    color: #eee;
    padding: 8px 10px;
    font-family: inherit;
    box-sizing: border-box;
    resize: vertical;
  }

  .toast {
    position: fixed;
    left: 50%;
    bottom: 24px;
    transform: translateX(-50%);
    background: #26272b;
    border: 1px solid #34353a;
    color: #eee;
    padding: 10px 18px;
    border-radius: 999px;
    font-size: 13px;
    box-shadow: 0 6px 20px rgba(0,0,0,0.4);
    z-index: 20;
  }

  @media (max-width: 640px) {
    .detail-grid { grid-template-columns: 1fr; }
  }
</style>
