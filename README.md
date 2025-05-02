// Utility Functions
function loadFromLocalStorage(key, defaultValue = []) {
    try {
        const data = localStorage.getItem(key);
        return data ? JSON.parse(data) : defaultValue;
    } catch (e) {
        console.error(`Error loading ${key} from localStorage:`, e);
        return defaultValue;
    }
}

function saveToLocalStorage(key, data) {
    try {
        localStorage.setItem(key, JSON.stringify(data));
    } catch (e) {
        console.error(`Error saving ${key} to localStorage:`, e);
        alert('Failed to save data. Check browser storage permissions.');
    }
}

function debounce(func, wait) {
    let timeout;
    return function (...args) {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
    };
}

// DataStore Class with Version History
class DataStore {
    constructor(key) {
        this.key = key;
        this.data = loadFromLocalStorage(key);
        this.history = loadFromLocalStorage(`${key}_history`, {});
    }

    add(item) {
        item.id = Date.now();
        item.order = this.data.length;
        this.data.push(item);
        this.save();
        if (this.key === 'emails' || this.key === 'notes') {
            this.history[item.id] = [{ ...item, timestamp: Date.now() }];
            this.saveHistory();
        }
    }

    delete(id) {
        this.data = this.data.filter(item => item.id !== id);
        if (this.history[id]) {
            delete this.history[id];
            this.saveHistory();
        }
        this.save();
    }

    get(id) {
        return this.data.find(item => item.id === id);
    }

    update(id, updatedItem) {
        const index = this.data.findIndex(item => item.id === id);
        if (index !== -1) {
            const oldItem = { ...this.data[index] };
            this.data[index] = { ...this.data[index], ...updatedItem };
            this.save();
            if ((this.key === 'emails' || this.key === 'notes') && updatedItem.title && updatedItem.content) {
                this.history[id] = this.history[id] || [];
                this.history[id].push({ ...this.data[index], timestamp: Date.now() });
                this.saveHistory();
            }
        }
    }

    save() {
        saveToLocalStorage(this.key, this.data);
    }

    saveHistory() {
        saveToLocalStorage(`${this.key}_history`, this.history);
    }

    getHistory(id) {
        return this.history[id] || [];
    }
}

// Initialize Data Stores
const emails = new DataStore('emails');
const contacts = new DataStore('contacts');
const todos = new DataStore('todos');
const notes = new DataStore('notes');

// Firebase Setup
let firebaseApp = null;
let firebaseDb = null;
function initializeFirebase(config) {
    try {
        firebaseApp = firebase.initializeApp(config);
        firebaseDb = firebase.database();
        syncWithFirebase();
    } catch (e) {
        console.error('Firebase initialization failed:', e);
        alert('Failed to initialize Firebase. Using local storage.');
    }
}

function syncWithFirebase() {
    if (!firebaseDb) return;
    ['emails', 'contacts', 'todos', 'notes'].forEach(key => {
        const ref = firebaseDb.ref(key);
        ref.on('value', snapshot => {
            const data = snapshot.val();
            if (data) {
                const store = window[key];
                store.data = Object.values(data);
                store.save();
                if (key === 'emails') renderEmails();
                if (key === 'contacts') {
                    renderContacts();
                    updateContactDropdowns();
                }
                if (key === 'todos') renderTodos();
                if (key === 'notes') renderNotes();
            }
        });
        window[key].save = function () {
            saveToLocalStorage(this.key, this.data);
            ref.set(this.data);
        };
    });
}

// Load Firebase Config
const firebaseConfig = loadFromLocalStorage('firebase_config', null);
if (firebaseConfig) {
    initializeFirebase(firebaseConfig);
}

// Theme Toggle
const body = document.body;
const themeToggle = document.getElementById('theme-toggle');
const currentTheme = loadFromLocalStorage('theme', 'light');
body.classList.add(currentTheme);
themeToggle.addEventListener('click', () => {
    body.classList.toggle('light');
    body.classList.toggle('dark');
    const newTheme = body.classList.contains('dark') ? 'dark' : 'light';
    saveToLocalStorage('theme', newTheme);
});

// Keyboard Shortcuts
document.addEventListener('keydown', e => {
    if (e.ctrlKey) {
        switch (e.key) {
            case 'e':
                e.preventDefault();
                switchTab('emails-tab');
                document.getElementById('email-title').focus();
                break;
            case 'c':
                e.preventDefault();
                switchTab('contacts-tab');
                document.getElementById('contact-name').focus();
                break;
            case 't':
                e.preventDefault();
                switchTab('todos-tab');
                document.getElementById('todo-title').focus();
                break;
            case 'n':
                e.preventDefault();
                switchTab('notes-tab');
                document.getElementById('note-title').focus();
                break;
            case '/':
                e.preventDefault();
                themeToggle.click();
                break;
        }
    }
});

// Tab Navigation
const tabs = document.querySelectorAll('nav button');
const contents = document.querySelectorAll('.tab-content');
tabs.forEach(tab => {
    tab.addEventListener('click', () => switchTab(tab.id));
});

function switchTab(tabId) {
    tabs.forEach(t => t.setAttribute('aria-selected', 'false'));
    contents.forEach(c => c.classList.remove('active'));
    const tab = document.getElementById(tabId);
    tab.setAttribute('aria-selected', 'true');
    document.getElementById(tab.getAttribute('aria-controls')).classList.add('active');
}

// Collapsible Sections
document.querySelectorAll('.collapse-btn').forEach(btn => {
    btn.addEventListener('click', () => {
        const target = document.getElementById(btn.dataset.target);
        target.classList.toggle('collapsed');
        btn.textContent = target.classList.contains('collapsed') ? '▶' : '▼';
    });
});

// Global Search
const globalSearch = document.getElementById('global-search');
globalSearch.addEventListener('input', debounce(() => {
    renderEmails();
    renderContacts();
    renderTodos();
    renderNotes();
}, 300));

// Emails
document.getElementById('email-form').addEventListener('submit', e => {
    e.preventDefault();
    const title = document.getElementById('email-title').value.trim();
    const category = document.getElementById('email-category').value.trim();
    const template = document.getElementById('email-template').value.trim();
    if (title && template) {
        emails.add({ title, category, template });
        e.target.reset();
        renderEmails();
        renderInsights();
    }
});

function renderEmail(email) {
    const li = document.createElement('li');
    li.innerHTML = `
        <input type="checkbox" class="bulk-select" data-id="${email.id}">
        ${email.title} (${email.category || 'Uncategorized'})
    `;
    const copyBtn = document.createElement('button');
    copyBtn.textContent = 'Copy';
    copyBtn.addEventListener('click', () => {
        const contact = email.contactId ? contacts.get(email.contactId) : null;
        const template = replaceTemplateVariables(email.template, contact);
        copyToClipboard(template, 'Email template copied!');
    });
    const sendBtn = document.createElement('button');
    sendBtn.textContent = 'Send';
    sendBtn.addEventListener('click', () => {
        const contact = email.contactId ? contacts.get(email.contactId) : null;
        const template = replaceTemplateVariables(email.template, contact);
        if (contact?.email) {
            window.location.href = `mailto:${contact.email}?subject=${encodeURIComponent(email.title)}&body=${encodeURIComponent(template)}`;
        } else {
            alert('No contact email linked.');
        }
    });
    const viewBtn = document.createElement('button');
    viewBtn.textContent = 'View';
    viewBtn.addEventListener('click', () => openModal('view', email.id));
    const editBtn = document.createElement('button');
    editBtn.textContent = 'Edit';
    editBtn.addEventListener('click', () => openModal('edit', email.id));
    const deleteBtn = document.createElement('button');
    deleteBtn.textContent = 'Delete';
    deleteBtn.addEventListener('click', () => {
        if (confirm(`Are you sure you want to delete "${email.title}"?`)) {
            emails.delete(email.id);
            renderEmails();
            renderInsights();
        }
    });
    li.append(copyBtn, sendBtn, viewBtn, editBtn, deleteBtn);
    return li;
}

function renderEmails() {
    const emailList = document.getElementById('email-list');
    emailList.innerHTML = '';
    const searchQuery = globalSearch.value.toLowerCase();
    const filteredEmails = emails.data.filter(email =>
        email.title.toLowerCase().includes(searchQuery) ||
        email.category?.toLowerCase().includes(searchQuery)
    );
    filteredEmails.forEach(email => emailList.appendChild(renderEmail(email)));
    updateBulkActions('emails');
}

let currentEmailId;
function openModal(mode, id) {
    currentEmailId = id;
    const email = emails.get(id);
    if (!email) return;
    const viewContent = document.getElementById('view-content');
    const editContent = document.getElementById('edit-content');
    const historyContent = document.getElementById('history-content');
    if (mode === 'view') {
        viewContent.style.display = 'block';
        editContent.style.display = 'none';
        historyContent.style.display = 'none';
        document.getElementById('view-title').textContent = email.title;
        document.getElementById('view-category').textContent = email.category ? `Category: ${email.category}` : '';
        document.getElementById('view-template').textContent = email.template;
    } else if (mode === 'edit') {
        viewContent.style.display = 'none';
        editContent.style.display = 'block';
        historyContent.style.display = 'none';
        document.getElementById('edit-title').value = email.title;
        document.getElementById('edit-category').value = email.category || '';
        document.getElementById('edit-template').value = email.template;
    } else if (mode === 'history') {
        viewContent.style.display = 'none';
        editContent.style.display = 'none';
        historyContent.style.display = 'block';
        renderHistory(id, 'emails');
    }
    document.getElementById('modal').style.display = 'flex';
}

function closeModal() {
    document.getElementById('modal').style.display = 'none';
}

document.getElementById('view-edit').addEventListener('click', () => openModal('edit', currentEmailId));
document.getElementById('view-history').addEventListener('click', () => openModal('history', currentEmailId));
document.getElementById('edit-save').addEventListener('click', () => {
    const title = document.getElementById('edit-title').value.trim();
    const category = document.getElementById('edit-category').value.trim();
    const template = document.getElementById('edit-template').value.trim();
    if (!title || !template) {
        alert('Title and template are required.');
        return;
    }
    emails.update(currentEmailId, { title, category, template });
    renderEmails();
    renderInsights();
    closeModal();
});
document.getElementById('edit-cancel').addEventListener('click', closeModal);
document.getElementById('view-copy').addEventListener('click', () => {
    const email = emails.get(currentEmailId);
    if (email) {
        const contact = email.contactId ? contacts.get(email.contactId) : null;
        const template = replaceTemplateVariables(email.template, contact);
        copyToClipboard(template, 'Email template copied!');
    }
});
document.getElementById('modal').addEventListener('click', e => {
    if (e.target === document.getElementById('modal')) closeModal();
});
document.getElementById('modal-close').addEventListener('click', closeModal);

// Contacts
function formatPhoneNumber(phone) {
    if (!phone) return 'No phone';
    const digits = phone.replace(/\D/g, '');
    if (digits.length === 10) {
        return `(${digits.slice(0, 3)}) ${digits.slice(3, 6)}-${digits.slice(6)}`;
    }
    return phone;
}

document.getElementById('contact-form').addEventListener('submit', e => {
    e.preventDefault();
    const name = document.getElementById('contact-name').value.trim();
    const email = document.getElementById('contact-email').value.trim();
    const phone = document.getElementById('contact-phone').value.trim();
    const address = document.getElementById('contact-address').value.trim();
    if (name) {
        contacts.add({ name, email, phone, address });
        e.target.reset();
        renderContacts();
        updateContactDropdowns();
    }
});

function renderContacts() {
    const contactList = document.getElementById('contact-list');
    const searchQuery = globalSearch.value.toLowerCase() || document.getElementById('contact-search').value.toLowerCase();
    contactList.innerHTML = '';
    const filteredContacts = contacts.data.filter(contact =>
        contact.name.toLowerCase().includes(searchQuery)
    );
    filteredContacts.forEach(contact => {
        const formattedPhone = formatPhoneNumber(contact.phone);
        const li = document.createElement('li');
        li.innerHTML = `
            <div class="tooltip" data-tooltip="Copy" data-copy="${contact.name || 'No name'}">${contact.name || 'No name'}</div><br>
            <div class="tooltip" data-tooltip="Copy" data-copy="${formattedPhone}">${formattedPhone}</div><br>
            <div class="tooltip" data-tooltip="Copy" data-copy="${contact.email || 'No email'}">${contact.email || 'No email'}</div><br>
            <div class="tooltip" data-tooltip="Copy" data-copy="${contact.address || 'No address'}">${contact.address || 'No address'}</div>
        `;
        contactList.appendChild(li);
    });
    addTooltipListeners();
}

document.getElementById('contact-search').addEventListener('input', renderContacts);

// To-Dos
function updateContactDropdowns() {
    const todoDropdown = document.getElementById('todo-contact');
    const noteDropdown = document.getElementById('note-contact');
    const options = `<option value="">None</option>` +
        contacts.data.map(contact => `<option value="${contact.id}">${contact.name}</option>`).join('');
    todoDropdown.innerHTML = options;
    noteDropdown.innerHTML = options;
}

document.getElementById('todo-form').addEventListener('submit', e => {
    e.preventDefault();
    const title = document.getElementById('todo-title').value.trim();
    const dueDate = document.getElementById('todo-due').value;
    const priority = document.getElementById('todo-priority').value;
    const status = document.getElementById('todo-status').value;
    const contactId = document.getElementById('todo-contact').value || null;
    if (title) {
        todos.add({ title, dueDate, priority, status, contactId });
        e.target.reset();
        renderTodos();
        renderCalendar();
        renderInsights();
    }
});

function renderTodos() {
    const todoList = document.getElementById('todo-list');
    todoList.innerHTML = '';
    const searchQuery = globalSearch.value.toLowerCase();
    let filteredTodos = todos.data.filter(todo =>
        todo.title.toLowerCase().includes(searchQuery)
    );
    const sortBy = document.getElementById('todo-sort').value;
    filteredTodos.sort((a, b) => {
        if (sortBy === 'priority') {
            const priorities = { High: 3, Medium: 2, Low: 1 };
            return priorities[b.priority] - priorities[a.priority];
        } else if (sortBy === 'status') {
            return b.status.localeCompare(a.status);
        } else {
            return (a.dueDate || '9999-12-31').localeCompare(b.dueDate || '9999-12-31');
        }
    });
    filteredTodos.forEach(todo => {
        const li = document.createElement('li');
        li.draggable = true;
        li.dataset.id = todo.id;
        li.classList.add('draggable');
        const contact = todo.contactId ? contacts.get(todo.contactId) : null;
        li.innerHTML = `
            <input type="checkbox" class="bulk-select" data-id="${todo.id}">
            ${todo.title} ${todo.dueDate ? `(Due: ${todo.dueDate})` : ''} 
            [${todo.priority}] [${todo.status}]
            ${contact ? `<span class="linked-contact" data-contact-id="${contact.id}">[${contact.name}]</span>` : ''}
        `;
        const markDoneBtn = document.createElement('button');
        markDoneBtn.textContent = 'Mark Done';
        markDoneBtn.addEventListener('click', () => {
            todos.update(todo.id, { status: 'Completed' });
            renderTodos();
            renderCalendar();
            renderInsights();
        });
        li.appendChild(markDoneBtn);
        todoList.appendChild(li);
    });
    addDragAndDrop(todoList, todos);
    updateBulkActions('todos');
    document.querySelectorAll('.linked-contact').forEach(span => {
        span.addEventListener('click', () => showContactTooltip(span.dataset.contactId));
    });
}

// Calendar View
document.getElementById('calendar-view-btn').addEventListener('click', () => {
    const calendarView = document.getElementById('calendar-view');
    calendarView.style.display = calendarView.style.display === 'none' ? 'block' : 'none';
    if (calendarView.style.display === 'block') renderCalendar();
});

function renderCalendar() {
    const calendarView = document.getElementById('calendar-view');
    calendarView.innerHTML = '';
    const today = new Date();
    const year = today.getFullYear();
    const month = today.getMonth();
    const firstDay = new Date(year, month, 1).getDay();
    const daysInMonth = new Date(year, month + 1, 0).getDate();
    const calendar = document.createElement('div');
    calendar.className = 'calendar';
    ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'].forEach(day => {
        const div = document.createElement('div');
        div.className = 'header';
        div.textContent = day;
        calendar.appendChild(div);
    });
    for (let i = 0; i < firstDay; i++) {
        calendar.appendChild(document.createElement('div'));
    }
    for (let day = 1; day <= daysInMonth; day++) {
        const div = document.createElement('div');
        const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
        const hasTask = todos.data.some(todo => todo.dueDate === dateStr);
        if (hasTask) div.className = 'task-day';
        div.textContent = day;
        calendar.appendChild(div);
    }
    calendarView.appendChild(calendar);
}

// Notes
document.getElementById('note-form').addEventListener('submit', e => {
    e.preventDefault();
    const title = document.getElementById('note-title').value.trim();
    const content = document.getElementById('note-content').value.trim();
    const contactId = document.getElementById('note-contact').value || null;
    if (title && content) {
        notes.add({ title, content, contactId });
        e.target.reset();
        renderNotes();
    }
});

function renderNotes() {
    const noteList = document.getElementById('note-list');
    noteList.innerHTML = '';
    const searchQuery = globalSearch.value.toLowerCase();
    const filteredNotes = notes.data.filter(note =>
        note.title.toLowerCase().includes(searchQuery) ||
        note.content.toLowerCase().includes(searchQuery)
    );
    filteredNotes.sort((a, b) => a.order - b.order);
    filteredNotes.forEach(note => {
        const li = document.createElement('li');
        li.draggable = true;
        li.dataset.id = note.id;
        li.classList.add('draggable');
        const contact = note.contactId ? contacts.get(note.contactId) : null;
        li.innerHTML = `
            <input type="checkbox" class="bulk-select" data-id="${note.id}">
            ${note.title}
            ${contact ? `<span class="linked-contact" data-contact-id="${contact.id}">[${contact.name}]</span>` : ''}
        `;
        const viewBtn = document.createElement('button');
        viewBtn.textContent = 'View';
        viewBtn.addEventListener('click', () => {
            alert(`Title: ${note.title}\nContent: ${note.content}`);
        });
        const historyBtn = document.createElement('button');
        historyBtn.textContent = 'History';
        historyBtn.addEventListener('click', () => openModal('history', note.id));
        li.append(viewBtn, historyBtn);
        noteList.appendChild(li);
    });
    addDragAndDrop(noteList, notes);
    updateBulkActions('notes');
    document.querySelectorAll('.linked-contact').forEach(span => {
        span.addEventListener('click', () => showContactTooltip(span.dataset.contactId));
    });
}

// Version History
function renderHistory(id, storeKey) {
    const historyList = document.getElementById('history-list');
    historyList.innerHTML = '';
    const store = window[storeKey];
    const history = store.getHistory(id);
    history.forEach((version, index) => {
        const li = document.createElement('li');
        li.innerHTML = `
            Version ${index + 1} - ${new Date(version.timestamp).toLocaleString()}
            <button>Revert</button>
        `;
        li.querySelector('button').addEventListener('click', () => {
            store.update(id, {
                title: version.title,
                category: version.category,
                template: version.template,
                content: version.content
            });
            if (storeKey === 'emails') renderEmails();
            if (storeKey === 'notes') renderNotes();
            closeModal();
        });
        historyList.appendChild(li);
    });
}

document.getElementById('history-close').addEventListener('click', closeModal);

// Pomodoro Timer
let timerInterval;
let timerSeconds = 25 * 60;
let isWorkSession = true;
let sessionsCompleted = loadFromLocalStorage('pomodoro_sessions', 0);
document.getElementById('timer-sessions').textContent = sessionsCompleted;

function updateTimerDisplay() {
    const minutes = Math.floor(timerSeconds / 60);
    const seconds = timerSeconds % 60;
    document.getElementById('timer-display').textContent = `${minutes}:${String(seconds).padStart(2, '0')}`;
}

document.getElementById('timer-start').addEventListener('click', () => {
    if (timerInterval) {
        clearInterval(timerInterval);
        timerInterval = null;
        document.getElementById('timer-start').textContent = 'Start';
    } else {
        timerInterval = setInterval(() => {
            timerSeconds--;
            updateTimerDisplay();
            if (timerSeconds <= 0) {
                clearInterval(timerInterval);
                timerInterval = null;
                document.getElementById('timer-start').textContent = 'Start';
                if (isWorkSession) {
                    sessionsCompleted++;
                    saveToLocalStorage('pomodoro_sessions', sessionsCompleted);
                    document.getElementById('timer-sessions').textContent = sessionsCompleted;
                    renderInsights();
                    timerSeconds = 5 * 60;
                    isWorkSession = false;
                    alert('Work session complete! Starting break.');
                } else {
                    timerSeconds = 25 * 60;
                    isWorkSession = true;
                    alert('Break complete! Starting work session.');
                }
                updateTimerDisplay();
            }
        }, 1000);
        document.getElementById('timer-start').textContent = 'Pause';
    }
});

document.getElementById('timer-reset').addEventListener('click', () => {
    clearInterval(timerInterval);
    timerInterval = null;
    timerSeconds = isWorkSession ? 25 * 60 : 5 * 60;
    updateTimerDisplay();
    document.getElementById('timer-start').textContent = 'Start';
});

// Insights
function renderInsights() {
    const tasksCompleted = todos.data.filter(todo => todo.status === 'Completed').length;
    const categories = {};
    emails.data.forEach(email => {
        const cat = email.category || 'Uncategorized';
        categories[cat] = (categories[cat] || 0) + 1;
    });
    document.getElementById('insights-tasks').textContent = tasksCompleted;
    document.getElementById('insights-categories').textContent = Object.entries(categories)
        .map(([cat, count]) => `${cat}: ${count}`)
        .join(', ');
    document.getElementById('insights-sessions').textContent = sessionsCompleted;
}

// Utility Functions
function copyToClipboard(text, message) {
    navigator.clipboard.writeText(text).then(() => {
        alert(message);
    }).catch(err => {
        console.error('Failed to copy text:', err);
    });
}

function addTooltipListeners() {
    document.querySelectorAll('.tooltip').forEach(tooltip => {
        tooltip.addEventListener('click', () => {
            const text = tooltip.getAttribute('data-copy');
            navigator.clipboard.writeText(text).then(() => {
                tooltip.setAttribute('data-tooltip', 'Copied');
                setTimeout(() => tooltip.setAttribute('data-tooltip', 'Copy'), 2000);
            });
        });
    });
}

function showContactTooltip(contactId) {
    const contact = contacts.get(contactId);
    alert(`Name: ${contact.name}\nPhone: ${formatPhoneNumber(contact.phone)}\nEmail: ${contact.email || 'No email'}\nAddress: ${contact.address || 'No address'}`);
}

function replaceTemplateVariables(template, contact) {
    let result = template;
    if (contact) {
        result = result.replace(/{contact\.name}/g, contact.name || '');
        result = result.replace(/{contact\.email}/g, contact.email || '');
        result = result.replace(/{contact\.phone}/g, contact.phone || '');
        result = result.replace(/{contact\.address}/g, contact.address || '');
    }
    result = result.replace(/{today}/g, new Date().toLocaleDateString());
    return result;
}

function addDragAndDrop(list, store) {
    list.addEventListener('dragstart', e => {
        const li = e.target.closest('li');
        if (!li) return;
        li.classList.add('dragging');
        e.dataTransfer.setData('text/plain', li.dataset.id);
    });

    list.addEventListener('dragend', e => {
        const li = e.target.closest('li');
        if (li) li.classList.remove('dragging');
    });

    list.addEventListener('dragover', e => {
        e.preventDefault();
        const li = e.target.closest('li');
        if (!li) return;
        const rect = li.getBoundingClientRect();
        const next = e.clientY > rect.top + rect.height / 2 ? li.nextSibling : li;
        list.insertBefore(document.querySelector('.dragging'), next);
    });

    list.addEventListener('drop', e => {
        e.preventDefault();
        const id = e.dataTransfer.getData('text/plain');
        const newOrder = Array.from(list.children).map(li => parseInt(li.dataset.id));
        store.data.forEach(item => {
            item.order = newOrder.indexOf(item.id);
        });
        store.data.sort((a, b) => a.order - b.order);
        store.save();
    });
}

// Bulk Actions
function updateBulkActions(type) {
    const checkboxes = document.querySelectorAll(`#${type}-list .bulk-select`);
    const deleteBtn = document.getElementById(`bulk-delete-${type}`);
    const selected = Array.from(checkboxes).filter(cb => cb.checked).map(cb => parseInt(cb.dataset.id));
    deleteBtn.disabled = selected.length === 0;
    deleteBtn.onclick = () => {
        if (confirm(`Delete ${selected.length} item(s)?`)) {
            selected.forEach(id => window[type].delete(id));
            if (type === 'emails') renderEmails();
            if (type === 'todos') {
                renderTodos();
                renderCalendar();
            }
            if (type === 'notes') renderNotes();
            renderInsights();
        }
    };
    if (type === 'emails') {
        const categorySelect = document.getElementById('bulk-category-emails');
        const applyBtn = document.getElementById('bulk-apply-category-emails');
        applyBtn.disabled = selected.length === 0;
        applyBtn.onclick = () => {
            const category = categorySelect.value;
            if (category) {
                selected.forEach(id => emails.update(id, { category }));
                renderEmails();
                renderInsights();
            }
        };
    }
    if (type === 'todos') {
        const statusSelect = document.getElementById('bulk-status-todos');
        const applyBtn = document.getElementById('bulk-apply-status-todos');
        applyBtn.disabled = selected.length === 0;
        applyBtn.onclick = () => {
            const status = statusSelect.value;
            if (status) {
                selected.forEach(id => todos.update(id, { status }));
                renderTodos();
                renderCalendar();
                renderInsights();
            }
        };
    }
}

// Export and Import
document.getElementById('export-btn').addEventListener('click', async () => {
    const data = {
        emails: emails.data,
        contacts: contacts.data,
        todos: todos.data.map(todo => ({
            ...todo,
            contactName: todo.contactId ? contacts.get(todo.contactId)?.name : null
        })),
        notes: notes.data.map(note => ({
            ...note,
            contactName: note.contactId ? contacts.get(note.contactId)?.name : null
        }))
    };
    const jsonData = JSON.stringify(data, null, 2);
    if (window.showSaveFilePicker) {
        try {
            const fileHandle = await window.showSaveFilePicker({
                suggestedName: 'productivity-data.json',
                types: [{ description: 'JSON File', accept: { 'application/json': ['.json'] } }],
            });
            const writable = await fileHandle.createWritable();
            await writable.write(jsonData);
            await writable.close();
            alert('Data exported successfully!');
        } catch (err) {
            console.error('Error during file save:', err);
            alert('Failed to export data. Using fallback method.');
            fallbackExport(jsonData);
        }
    } else {
        fallbackExport(jsonData);
    }
});

function fallbackExport(jsonData) {
    const blob = new Blob([jsonData], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'productivity-data.json';
    a.click();
    URL.revokeObjectURL(url);
}

document.getElementById('import-btn').addEventListener('click', () => {
    document.getElementById('import-file').click();
});

document.getElementById('import-file').addEventListener('change', e => {
    const file = e.target.files[0];
    if (file) {
        const reader = new FileReader();
        reader.onload = event => {
            try {
                const data = JSON.parse(event.target.result);
                if (data.emails) {
                    emails.data = data.emails;
                    emails.save();
                    renderEmails();
                }
                if (data.contacts) {
                    contacts.data = data.contacts;
                    contacts.save();
                    renderContacts();
                }
                if (data.todos) {
                    todos.data = data.todos;
                    todos.save();
                    renderTodos();
                    renderCalendar();
                }
                if (data.notes) {
                    notes.data = data.notes;
                    notes.save();
                    renderNotes();
                }
                updateContactDropdowns();
                renderInsights();
            } catch (error) {
                alert('Invalid JSON file. Please try again.');
            }
            e.target.value = '';
        };
        reader.readAsText(file);
    }
});

// Cloud Sync Configuration
document.getElementById('cloud-sync-btn').addEventListener('click', () => {
    document.getElementById('view-content').style.display = 'none';
    document.getElementById('edit-content').style.display = 'none';
    document.getElementById('history-content').style.display = 'none';
    document.getElementById('cloud-sync-content').style.display = 'block';
    document.getElementById('modal').style.display = 'flex';
});

document.getElementById('firebase-save').addEventListener('click', () => {
    try {
        const config = JSON.parse(document.getElementById('firebase-config').value);
        saveToLocalStorage('firebase_config', config);
        initializeFirebase(config);
        closeModal();
        alert('Firebase configured successfully!');
    } catch (e) {
        alert('Invalid Firebase configuration.');
    }
});

document.getElementById('firebase-cancel').addEventListener('click', closeModal);

// Initial Render
renderEmails();
renderContacts();
renderTodos();
renderNotes();
renderCalendar();
renderInsights();
updateContactDropdowns();