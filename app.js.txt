// StudySync - College Assignment Tracker Logic

const STORAGE_KEY = 'studysync_assignments_v1';
const THEME_KEY = 'studysync_theme';

let assignments = JSON.parse(localStorage.getItem(STORAGE_KEY)) || [];
let currentView = 'list';
let filters = { search: '', subject: 'ALL', priority: 'ALL', status: 'ALL' };

// DOM Elements
const listViewContainer = document.getElementById('listViewContainer');
const kanbanViewContainer = document.getElementById('kanbanViewContainer');
const emptyState = document.getElementById('emptyState');
const modal = document.getElementById('assignmentModal');
const assignmentForm = document.getElementById('assignmentForm');
const modalTitle = document.getElementById('modalTitle');
const assignmentIdInput = document.getElementById('assignmentId');

const searchInput = document.getElementById('searchInput');
const subjectFilter = document.getElementById('subjectFilter');
const priorityFilter = document.getElementById('priorityFilter');
const statusFilter = document.getElementById('statusFilter');

const viewListBtn = document.getElementById('viewListBtn');
const viewKanbanBtn = document.getElementById('viewKanbanBtn');

const themeToggleBtn = document.getElementById('themeToggleBtn');
const demoDataBtn = document.getElementById('demoDataBtn');
const exportBtn = document.getElementById('exportBtn');
const importInput = document.getElementById('importInput');
const clearAllBtn = document.getElementById('clearAllBtn');

const subjectColors = [
  { bg: 'bg-blue-50 dark:bg-blue-950/40', text: 'text-blue-700 dark:text-blue-300', border: 'border-blue-200 dark:border-blue-800' },
  { bg: 'bg-purple-50 dark:bg-purple-950/40', text: 'text-purple-700 dark:text-purple-300', border: 'border-purple-200 dark:border-purple-800' },
  { bg: 'bg-emerald-50 dark:bg-emerald-950/40', text: 'text-emerald-700 dark:text-emerald-300', border: 'border-emerald-200 dark:border-emerald-800' },
  { bg: 'bg-amber-50 dark:bg-amber-950/40', text: 'text-amber-700 dark:text-amber-300', border: 'border-amber-200 dark:border-amber-800' },
  { bg: 'bg-rose-50 dark:bg-rose-950/40', text: 'text-rose-700 dark:text-rose-300', border: 'border-rose-200 dark:border-rose-800' },
  { bg: 'bg-cyan-50 dark:bg-cyan-950/40', text: 'text-cyan-700 dark:text-cyan-300', border: 'border-cyan-200 dark:border-cyan-800' },
];

function getSubjectBadgeClass(subject) {
  let hash = 0;
  for (let i = 0; i < subject.length; i++) {
    hash = subject.charCodeAt(i) + ((hash << 5) - hash);
  }
  return subjectColors[Math.abs(hash) % subjectColors.length];
}

function getDeadlineStatus(dueDateStr, dueTimeStr, status) {
  if (status === 'Completed') {
    return {
      text: 'Completed',
      badgeClass: 'bg-emerald-100 dark:bg-emerald-950/60 text-emerald-700 dark:text-emerald-300 border border-emerald-200 dark:border-emerald-800',
      isUrgent: false
    };
  }

  const now = new Date();
  const dueDateTime = new Date(`${dueDateStr}T${dueTimeStr || '23:59'}:00`);
  const diffHours = (dueDateTime - now) / (1000 * 60 * 60);

  if (diffHours < 0) {
    const overdueDays = Math.ceil(Math.abs(diffHours) / 24);
    return {
      text: `⚠️ Overdue (${overdueDays}d ago)`,
      badgeClass: 'bg-rose-100 dark:bg-rose-950/60 text-rose-700 dark:text-rose-300 border border-rose-300 dark:border-rose-800 font-bold animate-pulse',
      isUrgent: true
    };
  } else if (diffHours <= 24) {
    const hoursLeft = Math.max(1, Math.round(diffHours));
    return {
      text: `🔥 Due in ${hoursLeft}h`,
      badgeClass: 'bg-rose-50 dark:bg-rose-950/50 text-rose-600 dark:text-rose-300 border border-rose-200 dark:border-rose-800 font-semibold',
      isUrgent: true
    };
  } else if (diffHours <= 48) {
    return {
      text: '⏳ Due tomorrow',
      badgeClass: 'bg-amber-100 dark:bg-amber-950/60 text-amber-700 dark:text-amber-300 border border-amber-200 dark:border-amber-800 font-semibold',
      isUrgent: true
    };
  } else {
    const daysLeft = Math.ceil(diffHours / 24);
    return {
      text: `📅 ${daysLeft} days left`,
      badgeClass: 'bg-slate-100 dark:bg-slate-800 text-slate-600 dark:text-slate-300 border border-slate-200 dark:border-slate-700',
      isUrgent: false
    };
  }
}

function updateStats() {
  const total = assignments.length;
  const completed = assignments.filter(a => a.status === 'Completed').length;
  const pending = assignments.filter(a => a.status !== 'Completed').length;

  const now = new Date();
  const urgent = assignments.filter(a => {
    if (a.status === 'Completed') return false;
    const due = new Date(`${a.dueDate}T${a.dueTime || '23:59'}:00`);
    const diffHours = (due - now) / (1000 * 60 * 60);
    return diffHours <= 48;
  }).length;

  const rate = total > 0 ? Math.round((completed / total) * 100) : 0;

  document.getElementById('statTotal').innerText = total;
  document.getElementById('statPending').innerText = pending;
  document.getElementById('statUrgent').innerText = urgent;
  document.getElementById('statRate').innerText = `${rate}%`;
  document.getElementById('statCompletedSub').innerText = `${completed} completed`;

  const circle = document.getElementById('progressCircle');
  const circumference = 2 * Math.PI * 22;
  circle.style.strokeDashoffset = circumference - (circumference * rate) / 100;
}

function updateSubjectDropdown() {
  const subjects = [...new Set(assignments.map(a => a.subject.trim()))].filter(Boolean);
  const currentVal = subjectFilter.value;
  
  let html = '<option value="ALL">All Subjects</option>';
  subjects.forEach(sub => {
    html += `<option value="${escapeHtml(sub)}">${escapeHtml(sub)}</option>`;
  });
  subjectFilter.innerHTML = html;
  if (subjects.includes(currentVal)) {
    subjectFilter.value = currentVal;
  }
}

function getFilteredAssignments() {
  return assignments.filter(item => {
    const matchesSearch = !filters.search || 
      item.title.toLowerCase().includes(filters.search.toLowerCase()) ||
      item.subject.toLowerCase().includes(filters.search.toLowerCase()) ||
      (item.notes && item.notes.toLowerCase().includes(filters.search.toLowerCase()));

    const matchesSubject = filters.subject === 'ALL' || item.subject === filters.subject;
    const matchesPriority = filters.priority === 'ALL' || item.priority === filters.priority;
    const matchesStatus = filters.status === 'ALL' || item.status === filters.status;

    return matchesSearch && matchesSubject && matchesPriority && matchesStatus;
  }).sort((a, b) => new Date(`${a.dueDate}T${a.dueTime}`) - new Date(`${b.dueDate}T${b.dueTime}`));
}

function renderUI() {
  updateStats();
  const filtered = getFilteredAssignments();

  if (filtered.length === 0) {
    listViewContainer.classList.add('hidden');
    kanbanViewContainer.classList.add('hidden');
    emptyState.classList.remove('hidden');
  } else {
    emptyState.classList.add('hidden');
    if (currentView === 'list') {
      renderListView(filtered);
      listViewContainer.classList.remove('hidden');
      kanbanViewContainer.classList.add('hidden');
    } else {
      renderKanbanView(filtered);
      listViewContainer.classList.add('hidden');
      kanbanViewContainer.classList.remove('hidden');
    }
  }

  if (window.lucide) lucide.createIcons();
}

function renderListView(items) {
  listViewContainer.innerHTML = items.map(assignment => {
    const isCompleted = assignment.status === 'Completed';
    const deadline = getDeadlineStatus(assignment.dueDate, assignment.dueTime, assignment.status);
    const subColor = getSubjectBadgeClass(assignment.subject);

    const priorityBadge = {
      High: '<span class="text-xs px-2.5 py-0.5 rounded-full bg-rose-100 dark:bg-rose-950/60 text-rose-700 dark:text-rose-300 font-semibold border border-rose-200 dark:border-rose-900">🔴 High</span>',
      Medium: '<span class="text-xs px-2.5 py-0.5 rounded-full bg-amber-100 dark:bg-amber-950/60 text-amber-700 dark:text-amber-300 font-semibold border border-amber-200 dark:border-amber-900">🟡 Medium</span>',
      Low: '<span class="text-xs px-2.5 py-0.5 rounded-full bg-emerald-100 dark:bg-emerald-950/60 text-emerald-700 dark:text-emerald-300 font-semibold border border-emerald-200 dark:border-emerald-900">🟢 Low</span>'
    }[assignment.priority] || '';

    return `
      <div class="assignment-card bg-white dark:bg-slate-900 p-4 sm:p-5 rounded-2xl border border-slate-200/90 dark:border-slate-800 shadow-sm flex flex-col sm:flex-row sm:items-center justify-between gap-4">
        <div class="flex items-start gap-3.5 flex-1 min-w-0">
          <button onclick="toggleAssignmentStatus('${assignment.id}')" aria-label="Toggle Complete" class="mt-1 flex-shrink-0 w-6 h-6 rounded-lg border-2 ${isCompleted ? 'bg-emerald-500 border-emerald-500 text-white' : 'border-slate-300 dark:border-slate-600 hover:border-indigo-500'} flex items-center justify-center transition">
            ${isCompleted ? '<i data-lucide="check" class="w-4 h-4"></i>' : ''}
          </button>

          <div class="min-w-0 flex-1">
            <div class="flex flex-wrap items-center gap-2 mb-1">
              <span class="text-xs px-2.5 py-0.5 rounded-md font-semibold ${subColor.bg} ${subColor.text} border ${subColor.border}">
                ${escapeHtml(assignment.subject)}
              </span>
              ${priorityBadge}
              <span class="text-xs px-2.5 py-0.5 rounded-full ${deadline.badgeClass}">
                ${deadline.text}
              </span>
            </div>

            <h3 class="text-base font-semibold text-slate-900 dark:text-white truncate ${isCompleted ? 'completed-title' : ''}">
              ${escapeHtml(assignment.title)}
            </h3>

            ${assignment.notes ? `<p class="text-xs text-slate-500 dark:text-slate-400 mt-1 line-clamp-1">${escapeHtml(assignment.notes)}</p>` : ''}
            
            <div class="flex items-center gap-3 mt-2 text-xs text-slate-400 dark:text-slate-500">
              <span class="flex items-center gap-1">
                <i data-lucide="calendar" class="w-3.5 h-3.5"></i>
                ${assignment.dueDate} at ${assignment.dueTime || '23:59'}
              </span>
              <span>•</span>
              <span class="capitalize font-medium ${assignment.status === 'Completed' ? 'text-emerald-500' : assignment.status === 'In Progress' ? 'text-amber-500' : 'text-slate-400'}">
                ${assignment.status}
              </span>
            </div>
          </div>
        </div>

        <div class="flex items-center sm:flex-col lg:flex-row gap-2 self-end sm:self-center">
          <button onclick="openEditModal('${assignment.id}')" class="p-2 rounded-xl text-slate-400 hover:text-indigo-600 dark:hover:text-indigo-400 hover:bg-slate-100 dark:hover:bg-slate-800 transition" title="Edit">
            <i data-lucide="edit-3" class="w-4 h-4"></i>
          </button>
          <button onclick="deleteAssignment('${assignment.id}')" class="p-2 rounded-xl text-slate-400 hover:text-rose-600 dark:hover:text-rose-400 hover:bg-slate-100 dark:hover:bg-slate-800 transition" title="Delete">
            <i data-lucide="trash-2" class="w-4 h-4"></i>
          </button>
        </div>
      </div>
    `;
  }).join('');
}

function renderKanbanView(items) {
  const pending = items.filter(a => a.status === 'Pending');
  const inProgress = items.filter(a => a.status === 'In Progress');
  const completed = items.filter(a => a.status === 'Completed');

  document.getElementById('kanbanCountPending').innerText = pending.length;
  document.getElementById('kanbanCountInProgress').innerText = inProgress.length;
  document.getElementById('kanbanCountCompleted').innerText = completed.length;

  const createCard = (a) => {
    const deadline = getDeadlineStatus(a.dueDate, a.dueTime, a.status);
    const subColor = getSubjectBadgeClass(a.subject);
    return `
      <div class="bg-white dark:bg-slate-800 p-4 rounded-xl border border-slate-200 dark:border-slate-700/80 shadow-sm space-y-2">
        <div class="flex items-center justify-between gap-1">
          <span class="text-[11px] px-2 py-0.5 rounded font-semibold ${subColor.bg} ${subColor.text} truncate max-w-[120px]">
            ${escapeHtml(a.subject)}
          </span>
          <span class="text-[10px] px-2 py-0.5 rounded-full ${deadline.badgeClass}">
            ${deadline.text}
          </span>
        </div>
        <h4 class="text-sm font-semibold text-slate-900 dark:text-white ${a.status === 'Completed' ? 'completed-title' : ''}">
          ${escapeHtml(a.title)}
        </h4>
        ${a.notes ? `<p class="text-xs text-slate-500 dark:text-slate-400 line-clamp-2">${escapeHtml(a.notes)}</p>` : ''}
        
        <div class="pt-2 flex items-center justify-between border-t border-slate-100 dark:border-slate-700/50 text-xs">
          <select onchange="updateAssignmentStatus('${a.id}', this.value)" class="text-[11px] py-1 px-1.5 rounded-lg border border-slate-200 dark:border-slate-600 bg-slate-50 dark:bg-slate-700 text-slate-700 dark:text-slate-200 focus:outline-none">
            <option value="Pending" ${a.status === 'Pending' ? 'selected' : ''}>To Do</option>
            <option value="In Progress" ${a.status === 'In Progress' ? 'selected' : ''}>In Progress</option>
            <option value="Completed" ${a.status === 'Completed' ? 'selected' : ''}>Completed</option>
          </select>
          <div class="flex items-center gap-1">
            <button onclick="openEditModal('${a.id}')" class="p-1 text-slate-400 hover:text-indigo-500 transition">
              <i data-lucide="edit-3" class="w-3.5 h-3.5"></i>
            </button>
            <button onclick="deleteAssignment('${a.id}')" class="p-1 text-slate-400 hover:text-rose-500 transition">
              <i data-lucide="trash-2" class="w-3.5 h-3.5"></i>
            </button>
          </div>
        </div>
      </div>
    `;
  };

  document.getElementById('kanbanListPending').innerHTML = pending.map(createCard).join('') || '<p class="text-xs text-slate-400 text-center py-4">No pending tasks</p>';
  document.getElementById('kanbanListInProgress').innerHTML = inProgress.map(createCard).join('') || '<p class="text-xs text-slate-400 text-center py-4">No tasks in progress</p>';
  document.getElementById('kanbanListCompleted').innerHTML = completed.map(createCard).join('') || '<p class="text-xs text-slate-400 text-center py-4">No completed tasks</p>';
}

function saveToStorage() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(assignments));
  updateSubjectDropdown();
  renderUI();
}

function openAddModal() {
  assignmentIdInput.value = '';
  modalTitle.innerText = 'Add New Assignment';
  assignmentForm.reset();
  
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  document.getElementById('formDueDate').value = tomorrow.toISOString().split('T')[0];
  document.getElementById('formDueTime').value = '23:59';
  
  modal.classList.remove('hidden');
}

function openEditModal(id) {
  const item = assignments.find(a => a.id === id);
  if (!item) return;

  assignmentIdInput.value = item.id;
  modalTitle.innerText = 'Edit Assignment';
  document.getElementById('formTitle').value = item.title;
  document.getElementById('formSubject').value = item.subject;
  document.getElementById('formPriority').value = item.priority;
  document.getElementById('formDueDate').value = item.dueDate;
  document.getElementById('formDueTime').value = item.dueTime || '23:59';
  document.getElementById('formStatus').value = item.status;
  document.getElementById('formNotes').value = item.notes || '';

  modal.classList.remove('hidden');
}

function closeModal() {
  modal.classList.add('hidden');
}

function toggleAssignmentStatus(id) {
  const item = assignments.find(a => a.id === id);
  if (!item) return;

  if (item.status === 'Completed') {
    item.status = 'Pending';
  } else {
    item.status = 'Completed';
    fireConfetti();
    showToast('🎉 Assignment marked as completed! Great job!');
  }
  saveToStorage();
}

function updateAssignmentStatus(id, newStatus) {
  const item = assignments.find(a => a.id === id);
  if (!item) return;

  item.status = newStatus;
  if (newStatus === 'Completed') {
    fireConfetti();
    showToast('🎉 Task marked as completed!');
  }
  saveToStorage();
}

function deleteAssignment(id) {
  if (confirm('Are you sure you want to delete this assignment?')) {
    assignments = assignments.filter(a => a.id !== id);
    saveToStorage();
    showToast('Assignment deleted', 'info');
  }
}

function fireConfetti() {
  if (typeof confetti === 'function') {
    confetti({
      particleCount: 60,
      spread: 70,
      origin: { y: 0.75 },
      colors: ['#6366f1', '#a855f7', '#10b981', '#f59e0b']
    });
  }
}

function showToast(message, type = 'success') {
  const container = document.getElementById('toastContainer');
  const toast = document.createElement('div');
  toast.className = `px-4 py-3 rounded-xl shadow-lg border text-sm font-semibold flex items-center gap-2 transform transition-all duration-300 translate-y-3 opacity-0 pointer-events-auto ${
    type === 'success' ? 'bg-slate-900 text-white dark:bg-white dark:text-slate-900 border-slate-800' : 'bg-rose-600 text-white'
  }`;
  toast.innerHTML = `<span>${message}</span>`;
  container.appendChild(toast);

  setTimeout(() => {
    toast.classList.remove('translate-y-3', 'opacity-0');
  }, 10);

  setTimeout(() => {
    toast.classList.add('opacity-0', 'translate-y-3');
    setTimeout(() => toast.remove(), 300);
  }, 2800);
}

function loadSampleData() {
  const today = new Date();
  const getDateStr = (offsetDays) => {
    const d = new Date(today);
    d.setDate(d.getDate() + offsetDays);
    return d.toISOString().split('T')[0];
  };

  assignments = [
    {
      id: 'demo-1',
      title: 'Operating Systems - CPU Scheduling Simulation',
      subject: 'Operating Systems',
      priority: 'High',
      dueDate: getDateStr(0),
      dueTime: '23:59',
      status: 'In Progress',
      notes: 'Implement Round Robin and SJF algorithms in C++. Submit PDF report with Gantt charts.'
    },
    {
      id: 'demo-2',
      title: 'Database Management - ER Diagram & Normalization (3NF)',
      subject: 'Database Systems',
      priority: 'High',
      dueDate: getDateStr(1),
      dueTime: '18:00',
      status: 'Pending',
      notes: 'Design schema for University Library Management system. Include all functional dependencies.'
    },
    {
      id: 'demo-3',
      title: 'Computer Networks - Packet Tracer Subnetting Lab',
      subject: 'Computer Networks',
      priority: 'Medium',
      dueDate: getDateStr(4),
      dueTime: '23:59',
      status: 'Pending',
      notes: 'Configure router IP addresses and RIP routing protocol. Upload .pkt file to portal.'
    },
    {
      id: 'demo-4',
      title: 'Mathematics III - Fourier Transform Problem Set 4',
      subject: 'Engineering Math',
      priority: 'Low',
      dueDate: getDateStr(7),
      dueTime: '17:00',
      status: 'Pending',
      notes: 'Solve textbook problems 4.1 to 4.15 on even functions.'
    },
    {
      id: 'demo-5',
      title: 'Software Engineering - Sprint 1 User Stories & SRS',
      subject: 'Software Engineering',
      priority: 'Medium',
      dueDate: getDateStr(-1),
      dueTime: '23:59',
      status: 'Completed',
      notes: 'Approved by project mentor. Sprint backlog finalized.'
    }
  ];

  saveToStorage();
  showToast('✨ Sample college assignments loaded!');
}

function escapeHtml(text) {
  if (!text) return '';
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

function initTheme() {
  const savedTheme = localStorage.getItem(THEME_KEY) || (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
  applyTheme(savedTheme);
}

function applyTheme(theme) {
  const html = document.documentElement;
  const icon = document.getElementById('themeIcon');
  if (theme === 'dark') {
    html.classList.add('dark');
    html.classList.remove('light');
    if (icon) icon.setAttribute('data-lucide', 'sun');
  } else {
    html.classList.remove('dark');
    html.classList.add('light');
    if (icon) icon.setAttribute('data-lucide', 'moon');
  }
  localStorage.setItem(THEME_KEY, theme);
  if (window.lucide) lucide.createIcons();
}

document.addEventListener('DOMContentLoaded', () => {
  initTheme();

  if (assignments.length === 0) {
    loadSampleData();
  } else {
    updateSubjectDropdown();
    renderUI();
  }

  themeToggleBtn.addEventListener('click', () => {
    const isDark = document.documentElement.classList.contains('dark');
    applyTheme(isDark ? 'light' : 'dark');
  });

  demoDataBtn.addEventListener('click', loadSampleData);
  document.getElementById('openAddModalBtn').addEventListener('click', openAddModal);
  document.getElementById('closeModalBtn').addEventListener('click', closeModal);
  document.getElementById('cancelModalBtn').addEventListener('click', closeModal);

  assignmentForm.addEventListener('submit', (e) => {
    e.preventDefault();
    const id = assignmentIdInput.value;
    const title = document.getElementById('formTitle').value.trim();
    const subject = document.getElementById('formSubject').value.trim();
    const priority = document.getElementById('formPriority').value;
    const dueDate = document.getElementById('formDueDate').value;
    const dueTime = document.getElementById('formDueTime').value || '23:59';
    const status = document.getElementById('formStatus').value;
    const notes = document.getElementById('formNotes').value.trim();

    if (id) {
      const index = assignments.findIndex(a => a.id === id);
      if (index !== -1) {
        assignments[index] = { ...assignments[index], title, subject, priority, dueDate, dueTime, status, notes };
        showToast('Assignment updated successfully!');
      }
    } else {
      const newAssignment = {
        id: 'task-' + Date.now(),
        title,
        subject,
        priority,
        dueDate,
        dueTime,
        status,
        notes
      };
      assignments.unshift(newAssignment);
      showToast('New assignment added!');
    }

    closeModal();
    saveToStorage();
  });

  searchInput.addEventListener('input', (e) => {
    filters.search = e.target.value;
    renderUI();
  });

  subjectFilter.addEventListener('change', (e) => {
    filters.subject = e.target.value;
    renderUI();
  });

  priorityFilter.addEventListener('change', (e) => {
    filters.priority = e.target.value;
    renderUI();
  });

  statusFilter.addEventListener('change', (e) => {
    filters.status = e.target.value;
    renderUI();
  });

  viewListBtn.addEventListener('click', () => {
    currentView = 'list';
    viewListBtn.className = 'px-2.5 py-1.5 rounded-lg text-xs font-semibold bg-white dark:bg-slate-700 text-indigo-600 dark:text-white shadow-sm flex items-center gap-1 transition';
    viewKanbanBtn.className = 'px-2.5 py-1.5 rounded-lg text-xs font-semibold text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-white flex items-center gap-1 transition';
    renderUI();
  });

  viewKanbanBtn.addEventListener('click', () => {
    currentView = 'kanban';
    viewKanbanBtn.className = 'px-2.5 py-1.5 rounded-lg text-xs font-semibold bg-white dark:bg-slate-700 text-indigo-600 dark:text-white shadow-sm flex items-center gap-1 transition';
    viewListBtn.className = 'px-2.5 py-1.5 rounded-lg text-xs font-semibold text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-white flex items-center gap-1 transition';
    renderUI();
  });

  exportBtn.addEventListener('click', () => {
    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(assignments, null, 2));
    const downloadAnchor = document.createElement('a');
    downloadAnchor.setAttribute("href", dataStr);
    downloadAnchor.setAttribute("download", `assignments_backup_${new Date().toISOString().split('T')[0]}.json`);
    document.body.appendChild(downloadAnchor);
    downloadAnchor.click();
    downloadAnchor.remove();
    showToast('Backup downloaded successfully!');
  });

  importInput.addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
      try {
        const imported = JSON.parse(event.target.result);
        if (Array.isArray(imported)) {
          assignments = imported;
          saveToStorage();
          showToast('Backup imported successfully!');
        } else {
          alert('Invalid backup file format.');
        }
      } catch (err) {
        alert('Could not read the JSON file.');
      }
    };
    reader.readAsText(file);
  });

  clearAllBtn.addEventListener('click', () => {
    if (confirm('Are you sure you want to clear all assignments? This cannot be undone.')) {
      assignments = [];
      saveToStorage();
      showToast('All assignments removed.', 'info');
    }
  });
});