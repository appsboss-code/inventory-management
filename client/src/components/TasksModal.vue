<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen" class="modal-overlay" @click="close">
        <div class="modal-container tasks-modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ t('tasks.title') }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Add Task Form -->
            <div class="task-form">
              <div class="form-row">
                <div class="form-group flex-1">
                  <label for="task-title">{{ t('tasks.taskTitle') }}</label>
                  <input
                    id="task-title"
                    v-model="newTask.title"
                    type="text"
                    :placeholder="t('tasks.taskTitlePlaceholder')"
                    class="task-input"
                    @keyup.enter="handleAddTask"
                  />
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label for="task-priority">{{ t('tasks.priority') }}</label>
                  <select
                    id="task-priority"
                    v-model="newTask.priority"
                    class="task-select"
                  >
                    <option value="high">{{ t('priority.high') }}</option>
                    <option value="medium">{{ t('priority.medium') }}</option>
                    <option value="low">{{ t('priority.low') }}</option>
                  </select>
                </div>

                <div class="form-group">
                  <label for="task-due-date">{{ t('tasks.dueDate') }}</label>
                  <input
                    id="task-due-date"
                    v-model="newTask.dueDate"
                    type="date"
                    class="task-input"
                  />
                </div>

                <div class="form-group-btn">
                  <button @click="handleAddTask" class="task-add-btn" :disabled="!newTask.title.trim() || !newTask.dueDate">
                    {{ t('tasks.addTask') }}
                  </button>
                </div>
              </div>
            </div>

            <div class="tasks-divider"></div>

            <!-- Tasks List -->
            <div v-if="sortedTasks.length === 0" class="no-tasks">
              {{ t('tasks.noTasks') }}
            </div>

            <div v-else class="tasks-list">
              <div
                v-for="task in sortedTasks"
                :key="task.id"
                class="task-item"
                :class="[`priority-${task.priority}`, { completed: task.status === 'completed' }]"
              >
                <div class="task-header">
                  <div class="task-check-title">
                    <input
                      type="checkbox"
                      :checked="task.status === 'completed'"
                      @change="$emit('toggle-task', task.id)"
                      class="task-checkbox"
                    />
                    <span class="task-title" @click="$emit('toggle-task', task.id)">{{ task.title }}</span>
                  </div>
                  <button @click="$emit('delete-task', task.id)" class="task-delete-btn" title="Delete task">
                    ×
                  </button>
                </div>

                <div class="task-footer">
                  <span class="priority-badge" :class="task.priority">
                    {{ translatePriority(task.priority) }}
                  </span>
                  <div class="task-due-date">
                    <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
                      <rect x="2" y="3" width="10" height="9" rx="1" stroke="currentColor" stroke-width="1.2"/>
                      <path d="M4.5 1.5V4.5M9.5 1.5V4.5M2 6H12" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
                    </svg>
                    {{ formatDueDate(task.dueDate) }}
                  </div>
                  <span class="status-badge" :class="getStatusClass(task.dueDate, task.status)">
                    {{ getStatusText(task.dueDate, task.status) }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">{{ t('profileDetails.close') }}</button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed } from 'vue'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'TasksModal',
  props: {
    isOpen: {
      type: Boolean,
      required: true
    },
    tasks: {
      type: Array,
      default: () => []
    }
  },
  emits: ['close', 'add-task', 'delete-task', 'toggle-task'],
  setup(props, { emit }) {
    const { t, currentLocale } = useI18n()
    const newTask = ref({
      title: '',
      priority: 'medium',
      dueDate: ''
    })

    const sortedTasks = computed(() => {
      // Don't sort - just return tasks in their current order (newest first)
      return [...props.tasks]
    })

    const close = () => {
      emit('close')
    }

    const handleAddTask = () => {
      if (newTask.value.title.trim() && newTask.value.dueDate) {
        emit('add-task', {
          title: newTask.value.title.trim(),
          priority: newTask.value.priority,
          dueDate: newTask.value.dueDate
        })
        newTask.value = {
          title: '',
          priority: 'medium',
          dueDate: ''
        }
      }
    }

    const formatDueDate = (dateString) => {
      const date = new Date(dateString)
      const today = new Date()
      today.setHours(0, 0, 0, 0)
      const dueDate = new Date(date)
      dueDate.setHours(0, 0, 0, 0)

      const diffTime = dueDate - today
      const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))

      const isJapanese = currentLocale.value === 'ja'

      if (diffDays === 0) return isJapanese ? '今日' : 'today'
      if (diffDays === 1) return isJapanese ? '明日' : 'tomorrow'
      if (diffDays === -1) return isJapanese ? '昨日' : 'yesterday'
      if (diffDays < 0) return isJapanese ? `${Math.abs(diffDays)}日前` : `${Math.abs(diffDays)} days ago`
      if (diffDays < 7) return isJapanese ? `${diffDays}日後` : `in ${diffDays} days`

      const locale = isJapanese ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, {
        month: 'short',
        day: 'numeric',
        year: date.getFullYear() !== today.getFullYear() ? 'numeric' : undefined
      })
    }

    const getStatusClass = (dueDate, status) => {
      if (status === 'completed') return 'completed'

      const today = new Date()
      today.setHours(0, 0, 0, 0)
      const due = new Date(dueDate)
      due.setHours(0, 0, 0, 0)

      const diffTime = due - today
      const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))

      if (diffDays < 0) return 'overdue'
      if (diffDays <= 1) return 'urgent'
      return 'upcoming'
    }

    const getStatusText = (dueDate, status) => {
      const isJapanese = currentLocale.value === 'ja'

      if (status === 'completed') return isJapanese ? '完了' : 'Completed'

      const statusClass = getStatusClass(dueDate, status)
      if (statusClass === 'overdue') return isJapanese ? '期限超過' : 'Overdue'
      if (statusClass === 'urgent') return isJapanese ? 'もうすぐ期限' : 'Due Soon'
      return isJapanese ? '予定' : 'Upcoming'
    }

    const translatePriority = (priority) => {
      const priorityMap = {
        'high': t('priority.high'),
        'medium': t('priority.medium'),
        'low': t('priority.low')
      }
      return priorityMap[priority] || priority
    }

    return {
      t,
      newTask,
      sortedTasks,
      close,
      handleAddTask,
      formatDueDate,
      getStatusClass,
      getStatusText,
      translatePriority
    }
  }
}
</script>

<style scoped>
/* z-index 1000 stays above the sidebar (--z-sidebar: 70) */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.5); /* slate-900 scrim */
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

/* Modal tier: radius-lg + shadow-overlay; widths from --modal-* tokens */
.modal-container {
  background: var(--color-surface);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-overlay);
  width: 90%;
  max-width: var(--modal-md); /* was 700px, nearest step */
  max-height: 85vh;
  display: flex;
  flex-direction: column;
}

.tasks-modal-container {
  max-width: var(--modal-lg);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--space-6) var(--space-7);
  border-bottom: 1px solid var(--color-border); /* was 2px */
}

.modal-title {
  font-size: 1.5rem;
  font-weight: var(--font-weight-semibold);
  color: var(--color-text);
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  color: var(--color-text-muted);
  cursor: pointer;
  padding: var(--space-2);
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius-sm);
  transition: background-color 0.2s ease, color 0.2s ease;
}

.close-button:hover {
  background: var(--color-hover);
  color: var(--color-text);
}

.modal-body {
  padding: var(--space-7);
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: var(--space-6) var(--space-7);
  border-top: 1px solid var(--color-border); /* was 2px */
  display: flex;
  justify-content: flex-end;
  gap: var(--space-4);
}

.btn-secondary {
  padding: var(--space-3) var(--space-6);
  background: var(--color-hover);
  color: var(--color-text-body);
  border: none;
  border-radius: var(--radius-sm);
  font-weight: var(--font-weight-semibold);
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.btn-secondary:hover {
  background: var(--color-border);
}

/* Task Form */
.task-form {
  background: var(--color-bg);
  border-radius: var(--radius-md);
  padding: var(--space-6);
  margin-bottom: var(--space-6);
}

.form-row {
  display: flex;
  flex-wrap: wrap; /* stack fields on narrow screens */
  gap: var(--space-4);
  margin-bottom: var(--space-4);
}

.form-row:last-child {
  margin-bottom: 0;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  flex: 1 1 160px;
}

.form-group.flex-1 {
  flex: 1 1 160px;
}

.form-group-btn {
  display: flex;
  align-items: flex-end;
}

label {
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-semibold);
  color: var(--color-text-body);
}

.task-input,
.task-select {
  padding: var(--space-3);
  border: 1px solid var(--color-border-strong);
  border-radius: var(--radius-sm);
  font-size: var(--font-size-sm); /* was 0.95rem; nearest step */
  transition: border-color 0.2s ease;
  font-family: inherit;
}

.task-input:focus,
.task-select:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-primary-tint);
}

.task-select {
  cursor: pointer;
  background: var(--color-surface);
}

/* Off-palette purple gradient (#667eea to #764ba2) replaced with the app primary */
.task-add-btn {
  padding: var(--space-3) var(--space-6); /* was 12px 28px; nearest step */
  background: var(--color-primary);
  color: var(--color-surface);
  border: none;
  border-radius: var(--radius-sm);
  font-weight: var(--font-weight-semibold);
  cursor: pointer;
  transition: background-color 0.2s ease, opacity 0.2s ease;
  white-space: nowrap;
  height: fit-content;
}

.task-add-btn:hover:not(:disabled) {
  background: var(--color-primary-strong);
}

.task-add-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.tasks-divider {
  height: 1px;
  background: var(--color-border);
  margin: var(--space-7) 0;
}

.no-tasks {
  text-align: center;
  padding: var(--space-8);
  color: var(--color-text-muted);
  font-size: var(--font-size-md);
  font-style: italic;
}

.tasks-list {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

.task-item {
  background: var(--color-surface);
  border: 1px solid var(--color-border); /* was 2px */
  border-radius: var(--radius-md);
  padding: var(--space-4) var(--space-5);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.task-item:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-card);
}

/* Priority is a full 4-sided tint (border + wash), replacing the 4px left stripe.
   The priority badge below still carries the label, so color is not the only cue. */
.task-item.priority-high { border-color: var(--color-danger-soft); background: var(--color-danger-wash); }
.task-item.priority-medium { border-color: var(--color-warning-soft); }
.task-item.priority-low { border-color: var(--color-info-soft); }

.task-item.completed {
  opacity: 0.6;
}

.task-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: var(--space-3);
  gap: var(--space-4);
}

.task-check-title {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  flex: 1;
}

.task-checkbox {
  width: 20px;
  height: 20px;
  cursor: pointer;
  accent-color: var(--color-primary);
  flex-shrink: 0;
}

.task-title {
  flex: 1;
  cursor: pointer;
  user-select: none;
  color: var(--color-text);
  font-size: var(--font-size-md);
  font-weight: var(--font-weight-semibold);
  line-height: 1.4;
}

.task-item.completed .task-title {
  text-decoration: line-through;
  color: var(--color-text-muted);
}

.task-delete-btn {
  width: 28px;
  height: 28px;
  background: var(--color-danger);
  color: var(--color-surface);
  border: none;
  border-radius: var(--radius-sm);
  font-size: var(--font-size-lg);
  line-height: 1;
  cursor: pointer;
  transition: background-color 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
}

.task-delete-btn:hover {
  background: var(--color-danger-ink);
}

.task-footer {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: var(--space-4);
}

/* Sentence case, floor of type scale is xs (was 0.688rem uppercase) */
.priority-badge {
  font-size: var(--font-size-xs);
  font-weight: var(--font-weight-semibold);
  padding: var(--space-1) var(--space-2); /* was 4px 10px; nearest step */
  border-radius: var(--radius-sm);
}

.priority-badge.high { background: var(--color-danger-soft); color: var(--color-danger-ink); }
.priority-badge.medium { background: var(--color-warning-soft); color: var(--color-warning-ink); }
.priority-badge.low { background: var(--color-info-soft); color: var(--color-info-ink); }

.task-due-date {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-size: var(--font-size-xs); /* was 0.813rem; nearest step */
  color: var(--color-text-muted);
}

.task-due-date svg {
  color: var(--color-text-muted);
}

.status-badge {
  font-size: var(--font-size-xs);
  font-weight: var(--font-weight-semibold);
  padding: var(--space-1) var(--space-2);
  border-radius: var(--radius-sm);
  margin-left: auto;
}

.status-badge.overdue { background: var(--color-danger-soft); color: var(--color-danger-ink); }
.status-badge.urgent { background: var(--color-warning-soft); color: var(--color-warning-ink); }
.status-badge.upcoming { background: var(--color-info-soft); color: var(--color-info-ink); }
.status-badge.completed { background: var(--color-success-soft); color: var(--color-success-ink); }

/* Modal transitions (global reduced-motion rule in base.css shortens these) */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.9);
}
</style>
