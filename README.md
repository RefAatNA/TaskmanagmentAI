<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Interactive Task Manager with Gemini AI, Projects & Enhanced Dashboard</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.2/dist/chart.umd.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.9.2/dist/confetti.browser.min.js"></script>
    <!-- Visualization & Content Choices: 
        - Task Data (ID, Desc, Status, Priority, AssignedTo, DueDate, Notes, Project): Goal: Inform/Organize. Method: JS objects in localStorage. Interaction: CRUD via modal. Justification: Core data structure. Library: Vanilla JS.
        - Project Field: Goal: Organize. Method: Text input in modal. Interaction: User input. Justification: Allows task grouping by project.
        - Kanban Board: Goal: Organize/Change. Method: HTML/Tailwind columns, JS cards (now showing Project). Interaction: Move tasks. Justification: Visual progression. Library: Vanilla JS.
        - List View: Goal: Organize/Inform. Method: HTML table (new Project column), JS rendering. Interaction: Filters (Status, Priority, new Project filter), Sort. Justification: Detailed tabular view. Library: Vanilla JS.
        - Dashboard - Tasks by Assignee Chart: Goal: Inform/Compare. Method: Chart.js bar chart. Interaction: Tooltips. Justification: Shows workload distribution. Library: Chart.js.
        - Dashboard - Tasks by Project Chart: Goal: Inform/Compare. Method: Chart.js bar chart. Interaction: Tooltips. Justification: Shows task concentration per project. Library: Chart.js.
        - Other features (Gemini, Confetti, etc.) remain as previously defined.
        - NO SVG graphics used. NO Mermaid JS used. -->
    <style>
        body { font-family: 'Inter', sans-serif; background-color: #F5F5F4; /* stone-100 */ }
        .kanban-column { min-height: 300px; }
        .task-card { transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out; }
        .task-card:hover { transform: translateY(-2px); box-shadow: 0 10px 15px -3px rgba(0,0,0,0.1), 0 4px 6px -2px rgba(0,0,0,0.05); }
        .modal { display: none; }
        .modal.active { display: flex; }
        .chart-container { position: relative; width: 100%; max-width: 500px; margin-left: auto; margin-right: auto; height: 300px; max-height: 350px; }
        @media (min-width: 768px) { .chart-container { height: 350px; max-height: 400px;} }
        .table-cell-truncate { max-width: 150px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        .custom-scrollbar::-webkit-scrollbar { width: 8px; height: 8px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: #e7e5e4; /* stone-200 */ border-radius: 10px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #a8a29e; /* stone-400 */ border-radius: 10px; }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover { background: #78716c; /* stone-500 */ }
        .tab-button.active { border-color: #0D9488; /* teal-600 */ color: #0D9488; background-color: #CCFBF1; /* teal-100 */}
        .tab-button { transition: all 0.2s ease-in-out; }
        .gemini-btn {
            background-color: #4F46E5; /* indigo-600 */
            color: white;
            padding: 0.3rem 0.6rem;
            font-size: 0.75rem; /* text-xs */
            border-radius: 0.375rem; /* rounded-md */
            transition: background-color 0.2s ease-in-out;
        }
        .gemini-btn:hover { background-color: #4338CA; /* indigo-700 */ }
        .gemini-btn:disabled { background-color: #A5B4FC; /* indigo-300 */ cursor: not-allowed; }
        .loader {
            border: 2px solid #f3f3f3; /* Light grey */
            border-top: 2px solid #4F46E5; /* Indigo */
            border-radius: 50%;
            width: 16px;
            height: 16px;
            animation: spin 1s linear infinite;
            display: inline-block;
            margin-left: 8px;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body class="text-stone-800">

    <div class="container mx-auto p-4 sm:p-6 lg:p-8 max-w-screen-2xl">
        <header class="mb-8">
            <h1 class="text-4xl font-bold text-stone-700 mb-2">Interactive Task Manager <span class="text-indigo-600 text-2xl align-middle">✨AI</span></h1>
            <p class="text-stone-500">Organize tasks by project, assign to team members, and gain insights with AI and an enhanced dashboard.</p>
            <div class="mt-6 flex flex-wrap gap-3 items-center">
                <button id="addTaskBtn" class="bg-teal-600 hover:bg-teal-700 text-white font-semibold py-2 px-4 rounded-lg shadow-md hover:shadow-lg transition-all duration-150 ease-in-out">
                    ➕ Add New Task
                </button>
                <div class="flex gap-1 p-1 bg-stone-200 rounded-lg">
                    <button id="kanbanViewBtn" class="tab-button active py-2 px-4 rounded-md text-sm font-medium text-stone-700 hover:bg-stone-300">Kanban Board</button>
                    <button id="listViewBtn" class="tab-button py-2 px-4 rounded-md text-sm font-medium text-stone-700 hover:bg-stone-300">List View</button>
                    <button id="dashboardViewBtn" class="tab-button py-2 px-4 rounded-md text-sm font-medium text-stone-700 hover:bg-stone-300">Dashboard</button>
                </div>
            </div>
        </header>

        <div id="taskModal" class="modal fixed inset-0 bg-black bg-opacity-50 items-center justify-center p-4 z-50 overflow-y-auto">
            <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-lg transform transition-all my-8">
                <div class="flex justify-between items-center">
                    <h2 id="modalTitle" class="text-2xl font-semibold mb-4 text-stone-700">Add New Task</h2>
                    <div id="geminiLoadingIndicatorModal" class="hidden items-center">
                        <span class="text-xs text-indigo-600 mr-2">Gemini is thinking...</span>
                        <div class="loader"></div>
                    </div>
                </div>
                <form id="taskForm">
                    <input type="hidden" id="taskId">
                    <div class="mb-1">
                        <label for="taskDescription" class="block text-sm font-medium text-stone-600 mb-1">Description</label>
                        <input type="text" id="taskDescription" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500" required>
                    </div>
                    <div class="mb-4 text-right">
                         <button type="button" id="smartDescriptionBtn" class="gemini-btn">✨ Smart Description</button>
                    </div>
                    
                    <div class="mb-4">
                        <label for="taskProject" class="block text-sm font-medium text-stone-600 mb-1">Project</label>
                        <input type="text" id="taskProject" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500" placeholder="e.g., Website Redesign">
                    </div>

                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="taskPriority" class="block text-sm font-medium text-stone-600 mb-1">Priority</label>
                            <select id="taskPriority" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500">
                                <option value="Low">Low</option>
                                <option value="Medium">Medium</option>
                                <option value="High">High</option>
                            </select>
                        </div>
                        <div>
                            <label for="taskStatusModal" class="block text-sm font-medium text-stone-600 mb-1">Status</label>
                            <select id="taskStatusModal" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500">
                                <option value="To Do">To Do</option>
                                <option value="In Progress">In Progress</option>
                                <option value="Blocked">Blocked</option>
                                <option value="Done">Done</option>
                            </select>
                        </div>
                    </div>
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="taskAssignedTo" class="block text-sm font-medium text-stone-600 mb-1">Assigned To</label>
                            <input type="text" id="taskAssignedTo" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500" placeholder="e.g., John Doe">
                        </div>
                        <div>
                            <label for="taskDueDate" class="block text-sm font-medium text-stone-600 mb-1">Due Date</label>
                            <input type="date" id="taskDueDate" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500">
                        </div>
                    </div>
                    <div class="mb-1">
                        <label for="taskNotes" class="block text-sm font-medium text-stone-600 mb-1">Notes</label>
                        <textarea id="taskNotes" rows="3" class="w-full p-2 border border-stone-300 rounded-md focus:ring-teal-500 focus:border-teal-500"></textarea>
                    </div>
                     <div class="mb-4 text-right">
                         <button type="button" id="suggestSubtasksBtn" class="gemini-btn">✨ Suggest Sub-tasks</button>
                    </div>
                    <div id="geminiResponseError" class="text-xs text-red-500 mb-3 hidden"></div>
                    <div class="flex justify-end gap-3">
                        <button type="button" id="cancelTaskBtn" class="px-4 py-2 text-stone-700 bg-stone-100 hover:bg-stone-200 rounded-lg transition-colors">Cancel</button>
                        <button type="submit" class="px-4 py-2 text-white bg-teal-600 hover:bg-teal-700 rounded-lg transition-colors shadow-sm">Save Task</button>
                    </div>
                </form>
            </div>
        </div>
        
        <main id="mainContent">
            <section id="kanbanView" class="mb-8">
                <h2 class="text-2xl font-semibold text-stone-700 mb-1">Kanban Board</h2>
                <p class="text-sm text-stone-500 mb-6">Move tasks with arrows. Click description to edit. AI & Project features in edit modal.</p>
                <div id="kanbanColumns" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                </div>
            </section>

            <section id="listView" class="hidden mb-8">
                <h2 class="text-2xl font-semibold text-stone-700 mb-1">Task List</h2>
                <p class="text-sm text-stone-500 mb-6">Detailed task view. Click description to edit & access AI/Project features.</p>
                <div class="mb-4 flex flex-wrap gap-4 items-center">
                    <div>
                        <label for="filterStatusList" class="text-sm font-medium text-stone-600 mr-2">Status:</label>
                        <select id="filterStatusList" class="p-2 border border-stone-300 rounded-md text-sm focus:ring-teal-500 focus:border-teal-500">
                            <option value="All">All</option>
                            <option value="To Do">To Do</option>
                            <option value="In Progress">In Progress</option>
                            <option value="Blocked">Blocked</option>
                            <option value="Done">Done</option>
                        </select>
                    </div>
                    <div>
                        <label for="filterPriorityList" class="text-sm font-medium text-stone-600 mr-2">Priority:</label>
                        <select id="filterPriorityList" class="p-2 border border-stone-300 rounded-md text-sm focus:ring-teal-500 focus:border-teal-500">
                            <option value="All">All</option>
                            <option value="High">High</option>
                            <option value="Medium">Medium</option>
                            <option value="Low">Low</option>
                        </select>
                    </div>
                    <div>
                        <label for="filterProjectList" class="text-sm font-medium text-stone-600 mr-2">Project:</label>
                        <select id="filterProjectList" class="p-2 border border-stone-300 rounded-md text-sm focus:ring-teal-500 focus:border-teal-500">
                            <option value="All">All Projects</option>
                        </select>
                    </div>
                     <div>
                        <label for="sortTasks" class="text-sm font-medium text-stone-600 mr-2">Sort by:</label>
                        <select id="sortTasks" class="p-2 border border-stone-300 rounded-md text-sm focus:ring-teal-500 focus:border-teal-500">
                            <option value="createdDateDesc">Created Date (Newest)</option>
                            <option value="createdDateAsc">Created Date (Oldest)</option>
                            <option value="dueDateAsc">Due Date (Soonest)</option>
                            <option value="dueDateDesc">Due Date (Latest)</option>
                            <option value="priority">Priority</option>
                            <option value="project">Project</option>
                        </select>
                    </div>
                </div>
                <div class="bg-white shadow-md rounded-lg overflow-x-auto custom-scrollbar">
                    <table class="min-w-full divide-y divide-stone-200">
                        <thead class="bg-stone-50">
                            <tr>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Description</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Project</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Status</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Priority</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Due Date</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Assigned To</th>
                                <th class="px-4 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">Actions</th>
                            </tr>
                        </thead>
                        <tbody id="taskListBody" class="bg-white divide-y divide-stone-200">
                        </tbody>
                    </table>
                </div>
            </section>

            <section id="dashboardView" class="hidden mb-8">
                <h2 class="text-2xl font-semibold text-stone-700 mb-1">Dashboard</h2>
                <p class="text-sm text-stone-500 mb-6">Insights into task distribution by status, priority, assignee, and project.</p>
                <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-sm font-medium text-stone-500">Total Tasks</h3>
                        <p id="totalTasksStat" class="text-3xl font-bold text-teal-600">0</p>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-sm font-medium text-stone-500">To Do</h3>
                        <p id="todoTasksStat" class="text-3xl font-bold text-rose-500">0</p>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-sm font-medium text-stone-500">In Progress</h3>
                        <p id="inprogressTasksStat" class="text-3xl font-bold text-amber-500">0</p>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-sm font-medium text-stone-500">Completed</h3>
                        <p id="doneTasksStat" class="text-3xl font-bold text-emerald-500">0</p>
                    </div>
                </div>
                <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-lg font-semibold text-stone-700 mb-4">Tasks by Status</h3>
                        <div class="chart-container">
                            <canvas id="statusChart"></canvas>
                        </div>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-lg font-semibold text-stone-700 mb-4">Tasks by Priority</h3>
                        <div class="chart-container">
                            <canvas id="priorityChart"></canvas>
                        </div>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-lg font-semibold text-stone-700 mb-4">Tasks by Assignee</h3>
                        <div class="chart-container">
                            <canvas id="assigneeChart"></canvas>
                        </div>
                    </div>
                    <div class="bg-white p-6 rounded-lg shadow-md">
                        <h3 class="text-lg font-semibold text-stone-700 mb-4">Tasks by Project</h3>
                        <div class="chart-container">
                            <canvas id="projectChart"></canvas>
                        </div>
                    </div>
                </div>
            </section>
        </main>

        <footer class="mt-12 text-center text-sm text-stone-500">
            <p>&copy; <span id="currentYear"></span> Interactive Task Manager. AI features by Gemini.</p>
        </footer>
    </div>

    <script>
        const KANBAN_COLUMNS = [
            { id: 'To Do', title: 'To Do', color: 'bg-rose-100', textColor: 'text-rose-700', borderColor: 'border-rose-400' },
            { id: 'In Progress', title: 'In Progress', color: 'bg-amber-100', textColor: 'text-amber-700', borderColor: 'border-amber-400' },
            { id: 'Blocked', title: 'Blocked', color: 'bg-red-100', textColor: 'text-red-700', borderColor: 'border-red-400' },
            { id: 'Done', title: 'Done', color: 'bg-emerald-100', textColor: 'text-emerald-700', borderColor: 'border-emerald-400' }
        ];

        let tasks = [];
        let currentEditTaskId = null;
        let statusChartInstance = null;
        let priorityChartInstance = null;
        let assigneeChartInstance = null;
        let projectChartInstance = null;


        const taskModal = document.getElementById('taskModal');
        const taskForm = document.getElementById('taskForm');
        const modalTitle = document.getElementById('modalTitle');
        const kanbanColumnsContainer = document.getElementById('kanbanColumns');
        const taskListBody = document.getElementById('taskListBody');

        const filterStatusListSelect = document.getElementById('filterStatusList');
        const filterPriorityListSelect = document.getElementById('filterPriorityList');
        const filterProjectListSelect = document.getElementById('filterProjectList');
        const sortTasksSelect = document.getElementById('sortTasks');

        const kanbanView = document.getElementById('kanbanView');
        const listView = document.getElementById('listView');
        const dashboardView = document.getElementById('dashboardView');
        
        const kanbanViewBtn = document.getElementById('kanbanViewBtn');
        const listViewBtn = document.getElementById('listViewBtn');
        const dashboardViewBtn = document.getElementById('dashboardViewBtn');

        const smartDescriptionBtn = document.getElementById('smartDescriptionBtn');
        const suggestSubtasksBtn = document.getElementById('suggestSubtasksBtn');
        const geminiLoadingIndicatorModal = document.getElementById('geminiLoadingIndicatorModal');
        const geminiResponseError = document.getElementById('geminiResponseError');


        function setActiveTab(activeBtn) {
            [kanbanViewBtn, listViewBtn, dashboardViewBtn].forEach(btn => btn.classList.remove('active'));
            activeBtn.classList.add('active');
        }

        function showView(viewToShow) {
            kanbanView.classList.add('hidden');
            listView.classList.add('hidden');
            dashboardView.classList.add('hidden');
            viewToShow.classList.remove('hidden');

            if (viewToShow === kanbanView) { setActiveTab(kanbanViewBtn); renderKanbanBoard(); }
            else if (viewToShow === listView) { setActiveTab(listViewBtn); renderListView(); }
            else if (viewToShow === dashboardView) { setActiveTab(dashboardViewBtn); renderDashboard(); }
        }
        
        kanbanViewBtn.addEventListener('click', () => showView(kanbanView));
        listViewBtn.addEventListener('click', () => showView(listView));
        dashboardViewBtn.addEventListener('click', () => showView(dashboardView));

        function openModal(task = null) {
            taskForm.reset();
            geminiResponseError.classList.add('hidden');
            geminiResponseError.textContent = '';
            if (task) {
                modalTitle.textContent = 'Edit Task';
                currentEditTaskId = task.id;
                document.getElementById('taskId').value = task.id;
                document.getElementById('taskDescription').value = task.description;
                document.getElementById('taskProject').value = task.project || '';
                document.getElementById('taskPriority').value = task.priority;
                document.getElementById('taskStatusModal').value = task.status;
                document.getElementById('taskAssignedTo').value = task.assignedTo || '';
                document.getElementById('taskDueDate').value = task.dueDate || '';
                document.getElementById('taskNotes').value = task.notes || '';
            } else {
                modalTitle.textContent = 'Add New Task';
                currentEditTaskId = null;
                document.getElementById('taskStatusModal').value = 'To Do';
            }
            taskModal.classList.add('active');
        }

        function closeModal() {
            taskModal.classList.remove('active');
            taskForm.reset();
            currentEditTaskId = null;
            geminiLoadingIndicatorModal.classList.add('hidden');
        }

        function saveTasksToLocalStorage() {
            localStorage.setItem('tasks', JSON.stringify(tasks));
        }

        function loadTasksFromLocalStorage() {
            const storedTasks = localStorage.getItem('tasks');
            if (storedTasks) {
                tasks = JSON.parse(storedTasks);
            }
        }

        function generateId() {
            return Date.now().toString(36) + Math.random().toString(36).substr(2);
        }
        
        function formatDate(dateString) {
            if (!dateString) return 'N/A';
            const date = new Date(dateString);
            if (isNaN(date.getTime())) return 'N/A';
            const [year, month, day] = dateString.split('-');
            const localDate = new Date(year, month - 1, day);
            return localDate.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
        }

        function isOverdue(dueDate) {
            if (!dueDate) return false;
            const today = new Date();
            today.setHours(0,0,0,0);
            const [year, month, day] = dueDate.split('-');
            const due = new Date(year, month - 1, day);
            due.setHours(0,0,0,0);
            return due < today;
        }

        taskForm.addEventListener('submit', (e) => {
            e.preventDefault();
            const description = document.getElementById('taskDescription').value.trim();
            if (!description) {
                alert('Task description cannot be empty.');
                return;
            }

            const taskData = {
                id: currentEditTaskId || generateId(),
                description: description,
                project: document.getElementById('taskProject').value.trim() || 'Default Project',
                priority: document.getElementById('taskPriority').value,
                status: document.getElementById('taskStatusModal').value,
                assignedTo: document.getElementById('taskAssignedTo').value.trim() || 'Unassigned',
                dueDate: document.getElementById('taskDueDate').value,
                notes: document.getElementById('taskNotes').value.trim(),
                createdDate: currentEditTaskId ? tasks.find(t => t.id === currentEditTaskId).createdDate : new Date().toISOString(),
                completedDate: null
            };
            
            const oldTask = currentEditTaskId ? tasks.find(t => t.id === currentEditTaskId) : null;
            const oldStatus = oldTask ? oldTask.status : null;

            if (taskData.status === 'Done') {
                taskData.completedDate = oldTask?.completedDate || new Date().toISOString(); // Preserve if already done, else set new
                if (oldStatus !== 'Done') {
                    triggerConfetti();
                }
            } else {
                 if (oldStatus === 'Done') taskData.completedDate = null; // Clear if moved out of done
                 else taskData.completedDate = oldTask?.completedDate; // Preserve if it was never done or already had a date (though this case is less likely)
            }

            if (currentEditTaskId) {
                tasks = tasks.map(t => t.id === currentEditTaskId ? taskData : t);
            } else {
                tasks.push(taskData);
            }
            
            saveTasksToLocalStorage();
            closeModal();
            refreshViews();
        });

        document.getElementById('addTaskBtn').addEventListener('click', () => openModal());
        document.getElementById('cancelTaskBtn').addEventListener('click', closeModal);

        function triggerConfetti() {
            if (typeof confetti === 'function') {
                confetti({ particleCount: 150, spread: 90, origin: { y: 0.6 }, zIndex: 10000 });
            }
        }
        
        function deleteTask(taskId) {
            if (confirm('Are you sure you want to delete this task?')) {
                tasks = tasks.filter(t => t.id !== taskId);
                saveTasksToLocalStorage();
                refreshViews();
            }
        }

        function moveTask(taskId, newStatus) {
            const task = tasks.find(t => t.id === taskId);
            if (task) {
                const oldStatus = task.status;
                task.status = newStatus;
                if (newStatus === 'Done') {
                    task.completedDate = task.completedDate || new Date().toISOString(); // Set if not already set
                    if (oldStatus !== 'Done') {
                         triggerConfetti();
                    }
                } else {
                    if (oldStatus === 'Done') {
                        task.completedDate = null;
                    }
                }
                saveTasksToLocalStorage();
                refreshViews();
            }
        }

        async function callGeminiAPI(promptText, targetElement, mode = 'replace') {
            geminiLoadingIndicatorModal.classList.remove('hidden');
            smartDescriptionBtn.disabled = true;
            suggestSubtasksBtn.disabled = true;
            geminiResponseError.classList.add('hidden');
            geminiResponseError.textContent = '';

            const apiKey = ""; 
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;
            
            const payload = {
                contents: [{ role: "user", parts: [{ text: promptText }] }]
            };

            try {
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                if (!response.ok) {
                    const errorData = await response.json();
                    console.error('Gemini API Error:', errorData);
                    throw new Error(`API Error: ${errorData.error?.message || response.statusText}`);
                }

                const result = await response.json();
                
                if (result.candidates && result.candidates.length > 0 &&
                    result.candidates[0].content && result.candidates[0].content.parts &&
                    result.candidates[0].content.parts.length > 0) {
                    const text = result.candidates[0].content.parts[0].text;
                    if (mode === 'replace') {
                        targetElement.value = text;
                    } else if (mode === 'append') {
                        targetElement.value += (targetElement.value ? '\n\n' : '') + "Suggested Sub-tasks:\n" + text;
                    }
                } else {
                    console.error('Gemini API Error: Unexpected response structure', result);
                    throw new Error('Failed to get a valid response from AI.');
                }

            } catch (error) {
                console.error('Error calling Gemini API:', error);
                geminiResponseError.textContent = `AI Error: ${error.message}. Please try again.`;
                geminiResponseError.classList.remove('hidden');
            } finally {
                geminiLoadingIndicatorModal.classList.add('hidden');
                smartDescriptionBtn.disabled = false;
                suggestSubtasksBtn.disabled = false;
            }
        }

        smartDescriptionBtn.addEventListener('click', () => {
            const taskDescriptionElement = document.getElementById('taskDescription');
            const currentDescription = taskDescriptionElement.value.trim();
            if (!currentDescription) {
                alert('Please enter a task title or brief idea first.');
                return;
            }
            const prompt = `Elaborate on this task title/idea: "${currentDescription}" into a concise and clear task description of 1-2 sentences.`;
            callGeminiAPI(prompt, taskDescriptionElement, 'replace');
        });

        suggestSubtasksBtn.addEventListener('click', () => {
            const taskDescription = document.getElementById('taskDescription').value.trim();
            const taskNotesElement = document.getElementById('taskNotes');
            if (!taskDescription) {
                alert('Please enter a task description first to suggest sub-tasks.');
                return;
            }
            const prompt = `Based on the task: "${taskDescription}", suggest 3-5 actionable sub-tasks to help complete it. List them clearly using bullet points (e.g., * Sub-task 1).`;
            callGeminiAPI(prompt, taskNotesElement, 'append');
        });

        function populateProjectFilter() {
            const uniqueProjects = [...new Set(tasks.map(task => task.project || 'Default Project'))].sort();
            filterProjectListSelect.innerHTML = '<option value="All">All Projects</option>'; // Reset
            uniqueProjects.forEach(project => {
                const option = document.createElement('option');
                option.value = project;
                option.textContent = project;
                filterProjectListSelect.appendChild(option);
            });
        }

        function renderKanbanBoard() {
            kanbanColumnsContainer.innerHTML = '';
            KANBAN_COLUMNS.forEach(column => {
                const columnDiv = document.createElement('div');
                columnDiv.className = `kanban-column p-4 rounded-lg shadow-sm ${column.color} min-h-[400px] flex flex-col`;
                columnDiv.innerHTML = `<h3 class="text-lg font-semibold ${column.textColor} mb-4 pb-2 border-b ${column.borderColor}">${column.title}</h3>`;
                
                const tasksInColumn = tasks.filter(task => task.status === column.id);
                const taskListDiv = document.createElement('div');
                taskListDiv.className = 'space-y-3 overflow-y-auto custom-scrollbar flex-grow';

                if (tasksInColumn.length === 0) {
                    taskListDiv.innerHTML = `<p class="text-sm text-stone-500 italic text-center py-4">No tasks here.</p>`;
                } else {
                    tasksInColumn.sort((a,b) => new Date(a.createdDate) - new Date(b.createdDate)).forEach(task => {
                        const taskCard = document.createElement('div');
                        taskCard.className = `task-card bg-white p-3 rounded-md shadow-sm border-l-4 ${task.priority === 'High' ? 'border-amber-500' : task.priority === 'Medium' ? 'border-sky-500' : 'border-stone-300'}`;
                        if (task.status === 'Done') taskCard.classList.add('opacity-70');
                        
                        let priorityColor = 'text-stone-500';
                        if (task.priority === 'High') priorityColor = 'text-amber-600 font-semibold';
                        if (task.priority === 'Medium') priorityColor = 'text-sky-600 font-semibold';

                        let overdueClass = '';
                        let overdueText = '';
                        if (task.status !== 'Done' && isOverdue(task.dueDate)) {
                            overdueClass = 'text-rose-600 font-semibold';
                            overdueText = ' <span class="text-xs">(Overdue)</span>';
                        }
                        
                        taskCard.innerHTML = `
                            <p class="font-medium text-stone-800 break-words ${task.status === 'Done' ? 'line-through' : ''} cursor-pointer" title="Click to edit">${task.description}</p>
                            <p class="text-xs text-indigo-600 mt-1">Project: ${task.project || 'Default Project'}</p>
                            <p class="text-xs ${priorityColor} mt-1">Priority: ${task.priority}</p>
                            <p class="text-xs ${overdueClass} text-stone-500 mt-1">Due: ${formatDate(task.dueDate)}${overdueText}</p>
                            ${task.assignedTo && task.assignedTo !== 'Unassigned' ? `<p class="text-xs text-stone-500 mt-1">To: ${task.assignedTo}</p>` : ''}
                            <div class="mt-3 flex justify-between items-center text-xs">
                                <div>
                                    ${ KANBAN_COLUMNS.findIndex(c => c.id === task.status) > 0 ? `<button class="move-task-btn p-1 hover:bg-stone-200 rounded" data-task-id="${task.id}" data-direction="prev" title="Move Left">◀</button>` : `<span class="p-1 text-stone-300">◀</span>`}
                                    ${ KANBAN_COLUMNS.findIndex(c => c.id === task.status) < KANBAN_COLUMNS.length - 1 ? `<button class="move-task-btn p-1 hover:bg-stone-200 rounded" data-task-id="${task.id}" data-direction="next" title="Move Right">▶</button>` : `<span class="p-1 text-stone-300">▶</span>`}
                                </div>
                                <button class="delete-task-btn text-rose-500 hover:text-rose-700 p-1 hover:bg-rose-100 rounded" data-task-id="${task.id}" title="Delete Task">🗑️</button>
                            </div>
                        `;
                        taskCard.querySelector('p.font-medium').addEventListener('click', () => openModal(task));
                        taskListDiv.appendChild(taskCard);
                    });
                }
                columnDiv.appendChild(taskListDiv);
                kanbanColumnsContainer.appendChild(columnDiv);
            });

            document.querySelectorAll('.delete-task-btn').forEach(btn => btn.addEventListener('click', (e) => deleteTask(e.currentTarget.dataset.taskId)));
            document.querySelectorAll('.move-task-btn').forEach(btn => btn.addEventListener('click', (e) => {
                const taskId = e.currentTarget.dataset.taskId;
                const direction = e.currentTarget.dataset.direction;
                const task = tasks.find(t => t.id === taskId);
                const currentIndex = KANBAN_COLUMNS.findIndex(c => c.id === task.status);
                const newIndex = direction === 'next' ? currentIndex + 1 : currentIndex - 1;
                if (newIndex >= 0 && newIndex < KANBAN_COLUMNS.length) {
                    moveTask(taskId, KANBAN_COLUMNS[newIndex].id);
                }
            }));
        }

        function renderListView() {
            taskListBody.innerHTML = '';
            populateProjectFilter(); 
            let filteredTasks = [...tasks];

            const statusFilter = filterStatusListSelect.value;
            if (statusFilter !== 'All') {
                filteredTasks = filteredTasks.filter(task => task.status === statusFilter);
            }

            const priorityFilter = filterPriorityListSelect.value;
            if (priorityFilter !== 'All') {
                filteredTasks = filteredTasks.filter(task => task.priority === priorityFilter);
            }

            const projectFilter = filterProjectListSelect.value;
            if (projectFilter !== 'All') {
                filteredTasks = filteredTasks.filter(task => (task.project || 'Default Project') === projectFilter);
            }
            
            const sortOption = sortTasksSelect.value;
            switch(sortOption) {
                case 'createdDateDesc': filteredTasks.sort((a,b) => new Date(b.createdDate) - new Date(a.createdDate)); break;
                case 'createdDateAsc': filteredTasks.sort((a,b) => new Date(a.createdDate) - new Date(b.createdDate)); break;
                case 'dueDateAsc': filteredTasks.sort((a,b) => (a.dueDate && b.dueDate ? new Date(a.dueDate) - new Date(b.dueDate) : a.dueDate ? -1 : 1)); break;
                case 'dueDateDesc': filteredTasks.sort((a,b) => (a.dueDate && b.dueDate ? new Date(b.dueDate) - new Date(a.dueDate) : a.dueDate ? -1 : 1)); break;
                case 'priority': 
                    const priorityOrder = { 'High': 1, 'Medium': 2, 'Low': 3 };
                    filteredTasks.sort((a,b) => priorityOrder[a.priority] - priorityOrder[b.priority]);
                    break;
                case 'project':
                    filteredTasks.sort((a,b) => (a.project || 'Default Project').localeCompare(b.project || 'Default Project'));
                    break;
            }

            if (filteredTasks.length === 0) {
                taskListBody.innerHTML = `<tr><td colspan="7" class="text-center py-4 text-stone-500 italic">No tasks match your filters.</td></tr>`;
                return;
            }

            filteredTasks.forEach(task => {
                const row = taskListBody.insertRow();
                row.className = `${task.status === 'Done' ? 'bg-stone-50 opacity-70' : 'bg-white'} hover:bg-stone-50 transition-colors`;
                
                let priorityClass = '';
                if (task.priority === 'High') priorityClass = 'text-amber-600 font-semibold';
                if (task.priority === 'Medium') priorityClass = 'text-sky-600';

                let overdueClass = '';
                if (task.status !== 'Done' && isOverdue(task.dueDate)) {
                    overdueClass = 'text-rose-600 font-semibold';
                }

                row.innerHTML = `
                    <td class="px-4 py-3 whitespace-nowrap">
                        <div class="text-sm text-stone-900 table-cell-truncate cursor-pointer hover:text-teal-600 ${task.status === 'Done' ? 'line-through' : ''}" title="${task.description}\nClick to edit">${task.description}</div>
                        <div class="text-xs text-stone-500">ID: ${task.id.substring(0,8)}...</div>
                    </td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm text-indigo-600 table-cell-truncate" title="${task.project || 'Default Project'}">${task.project || 'Default Project'}</td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm text-stone-500">${task.status}</td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm ${priorityClass}">${task.priority}</td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm ${overdueClass} text-stone-500">${formatDate(task.dueDate)}</td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm text-stone-500 table-cell-truncate" title="${task.assignedTo || 'Unassigned'}">${task.assignedTo || 'Unassigned'}</td>
                    <td class="px-4 py-3 whitespace-nowrap text-sm font-medium">
                        <button class="edit-task-list-btn text-teal-600 hover:text-teal-800 mr-2" data-task-id="${task.id}" title="Edit">✏️</button>
                        <button class="delete-task-btn text-rose-500 hover:text-rose-700" data-task-id="${task.id}" title="Delete">🗑️</button>
                    </td>
                `;
                row.querySelector('.table-cell-truncate.cursor-pointer').addEventListener('click', () => openModal(task));
            });
            document.querySelectorAll('.delete-task-btn').forEach(btn => btn.addEventListener('click', (e) => deleteTask(e.currentTarget.dataset.taskId)));
            document.querySelectorAll('.edit-task-list-btn').forEach(btn => btn.addEventListener('click', (e) => {
                const taskToEdit = tasks.find(t => t.id === e.currentTarget.dataset.taskId);
                if (taskToEdit) openModal(taskToEdit);
            }));
        }
        
        filterStatusListSelect.addEventListener('change', renderListView);
        filterPriorityListSelect.addEventListener('change', renderListView);
        filterProjectListSelect.addEventListener('change', renderListView);
        sortTasksSelect.addEventListener('change', renderListView);

        function getGroupedData(key) {
            const grouped = tasks.reduce((acc, task) => {
                const groupValue = task[key] || (key === 'project' ? 'Default Project' : 'Unassigned');
                acc[groupValue] = (acc[groupValue] || 0) + 1;
                return acc;
            }, {});
            return Object.entries(grouped).sort((a,b) => b[1] - a[1]); // Sort by count desc
        }
        
        function renderDashboard() {
            document.getElementById('totalTasksStat').textContent = tasks.length;
            const todoCount = tasks.filter(t => t.status === 'To Do').length;
            const inprogressCount = tasks.filter(t => t.status === 'In Progress').length;
            const doneCount = tasks.filter(t => t.status === 'Done').length;

            document.getElementById('todoTasksStat').textContent = todoCount;
            document.getElementById('inprogressTasksStat').textContent = inprogressCount;
            document.getElementById('doneTasksStat').textContent = doneCount;

            const statusCtx = document.getElementById('statusChart').getContext('2d');
            const priorityCtx = document.getElementById('priorityChart').getContext('2d');
            const assigneeCtx = document.getElementById('assigneeChart').getContext('2d');
            const projectCtx = document.getElementById('projectChart').getContext('2d');

            const statusData = KANBAN_COLUMNS.map(col => tasks.filter(t => t.status === col.id).length);
            const priorityData = ['High', 'Medium', 'Low'].map(p => tasks.filter(t => t.priority === p).length);
            
            const assigneeGrouped = getGroupedData('assignedTo');
            const assigneeLabels = assigneeGrouped.map(item => item[0]);
            const assigneeCounts = assigneeGrouped.map(item => item[1]);

            const projectGrouped = getGroupedData('project');
            const projectLabels = projectGrouped.map(item => item[0]);
            const projectCounts = projectGrouped.map(item => item[1]);

            const chartColors = ['#6EE7B7', '#FBBF24', '#F87171', '#60A5FA', '#A78BFA', '#F472B6', '#34D399', '#F59E0B', '#EF4444', '#3B82F6'];

            if (statusChartInstance) statusChartInstance.destroy();
            statusChartInstance = new Chart(statusCtx, {
                type: 'doughnut',
                data: {
                    labels: KANBAN_COLUMNS.map(col => col.title),
                    datasets: [{
                        label: 'Tasks by Status',
                        data: statusData,
                        backgroundColor: [ '#FECACA', '#FDE68A', '#FCA5A5', '#A7F3D0'], 
                        borderColor: [ '#F87171', '#FBBF24', '#EF4444', '#34D399'], 
                        borderWidth: 1
                    }]
                },
                options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { position: 'bottom' } } }
            });

            if (priorityChartInstance) priorityChartInstance.destroy();
            priorityChartInstance = new Chart(priorityCtx, {
                type: 'bar',
                data: {
                    labels: ['High', 'Medium', 'Low'],
                    datasets: [{
                        label: 'Tasks by Priority',
                        data: priorityData,
                        backgroundColor: ['#FBBF24', '#38BDF8', '#A8A29E'], 
                        borderColor: ['#F59E0B', '#0EA5E9', '#78716C'], 
                        borderWidth: 1
                    }]
                },
                options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } }, scales: { y: { beginAtZero: true, ticks: { stepSize: 1 } } } }
            });

            if (assigneeChartInstance) assigneeChartInstance.destroy();
            assigneeChartInstance = new Chart(assigneeCtx, {
                type: 'bar',
                data: {
                    labels: assigneeLabels,
                    datasets: [{
                        label: 'Tasks by Assignee',
                        data: assigneeCounts,
                        backgroundColor: chartColors.slice(0, assigneeLabels.length),
                        borderWidth: 1
                    }]
                },
                options: { responsive: true, maintainAspectRatio: false, indexAxis: 'y', plugins: { legend: { display: false } }, scales: { x: { beginAtZero: true, ticks: { stepSize: 1 } } } }
            });

            if (projectChartInstance) projectChartInstance.destroy();
            projectChartInstance = new Chart(projectCtx, {
                type: 'bar',
                data: {
                    labels: projectLabels,
                    datasets: [{
                        label: 'Tasks by Project',
                        data: projectCounts,
                        backgroundColor: chartColors.slice(0, projectLabels.length).reverse(), // Use different part of colors
                        borderWidth: 1
                    }]
                },
                options: { responsive: true, maintainAspectRatio: false, indexAxis: 'y', plugins: { legend: { display: false } }, scales: { x: { beginAtZero: true, ticks: { stepSize: 1 } } } }
            });
        }
        
        function refreshViews() {
            populateProjectFilter(); // Keep project filter up-to-date
            if (!kanbanView.classList.contains('hidden')) renderKanbanBoard();
            if (!listView.classList.contains('hidden')) renderListView();
            if (!dashboardView.classList.contains('hidden')) renderDashboard();
        }

        document.getElementById('currentYear').textContent = new Date().getFullYear();

        loadTasksFromLocalStorage();
        if (tasks.length === 0) { 
            tasks = [
                {id: generateId(), description: "Design homepage mockups", project: "Website Redesign", priority: "High", status: "To Do", assignedTo: "Alice", dueDate: new Date(Date.now() + 3 * 24*60*60*1000).toISOString().split('T')[0], notes: "Focus on modern UI", createdDate: new Date().toISOString(), completedDate: null},
                {id: generateId(), description: "Develop API for user auth", project: "Alpha Feature", priority: "High", status: "In Progress", assignedTo: "Bob", dueDate: new Date(Date.now() + 7 * 24*60*60*1000).toISOString().split('T')[0], notes: "JWT based", createdDate: new Date(Date.now() - 1 * 24*60*60*1000).toISOString(), completedDate: null},
                {id: generateId(), description: "Write user docs for API", project: "Alpha Feature", priority: "Medium", status: "To Do", assignedTo: "Carol", dueDate: new Date(Date.now() + 10 * 24*60*60*1000).toISOString().split('T')[0], notes: "", createdDate: new Date(Date.now() - 2 * 24*60*60*1000).toISOString(), completedDate: null},
                {id: generateId(), description: "Review budget Q3", project: "Admin", priority: "Low", status: "Done", assignedTo: "David", dueDate: new Date(Date.now() - 5 * 24*60*60*1000).toISOString().split('T')[0], notes: "Approved", createdDate: new Date(Date.now() - 10 * 24*60*60*1000).toISOString(), completedDate: new Date(Date.now() - 4 * 24*60*60*1000).toISOString()},
                {id: generateId(), description: "Client meeting for Website Redesign", project: "Website Redesign", priority: "High", status: "In Progress", assignedTo: "Alice", dueDate: new Date(Date.now() + 1 * 24*60*60*1000).toISOString().split('T')[0], notes: "Prepare presentation", createdDate: new Date(Date.now() - 0.5 * 24*60*60*1000).toISOString(), completedDate: null},
            ];
            saveTasksToLocalStorage();
        }
        showView(kanbanView); 
    </script>
</body>
</html>
