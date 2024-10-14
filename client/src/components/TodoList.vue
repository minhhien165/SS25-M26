<template>
  <div class="task-manager">
    <h2>Quản lý công việc</h2>

    <div class="add-task">
      <input
        type="text"
        placeholder="Nhập tên công việc"
        v-model="inputValue"
      />
      <button class="add-task-btn" @click="addJob">Thêm công việc</button>
    </div>
    <p v-show="errorAdd" style="color: red">
      Tên công việc không được phép để trống hoặc trùng!
    </p>

    <div class="task-filters">
      <button
        class="filter"
        :class="{ active: filter === 'all' }"
        @click="setFilter('all')"
      >
        Tất cả
      </button>
      <button
        class="filter"
        :class="{ active: filter === 'completed' }"
        @click="setFilter('completed')"
      >
        Hoàn thành
      </button>
      <button
        class="filter"
        :class="{ active: filter === 'in-progress' }"
        @click="setFilter('in-progress')"
      >
        Đang thực hiện
      </button>
    </div>

    <div v-if="isLoading" class="loading-spinner">Đang tải công việc...</div>

    <ul v-else class="task-list">
      <li v-for="(job, index) in filteredJobs" :key="index" class="task-item">
        <input type="checkbox" v-model="job.status" @click="handleCheck(job)" />
        <span :class="{ completed: job.status }">{{ job.name }}</span>
        <div class="task-actions">
          <button class="edit-task" @click="handleFormEdit(job)">
            &#9998;
          </button>
          <button class="delete-task" @click="handleFormDelete(job)">
            &#128465;
          </button>
        </div>
      </li>
    </ul>

    <div class="action-buttons">
      <button class="clear-completed" @click="handleUpdateIncompleteJobs">
        Xóa công việc hoàn thành
      </button>
      <button class="clear-all" @click="handleDeleteAll">
        Xóa tất cả công việc
      </button>
    </div>
    <div class="modal" v-show="isVisibleDelete">
      <div class="modal-content">
        <h2>Xác nhận</h2>
        <p>
          Bạn có chắc chắn muốn xóa công việc ->
          <b>{{ getJob.name }} - </b> không?
        </p>
        <div class="btn-delete">
          <button @click="onCancel">Hủy</button>
          <button @click="deleteTask">Xóa</button>
        </div>
      </div>
    </div>
    <div class="modal" v-show="isVisibleUpdate">
      <div class="modal-content">
        <h2>Chỉnh sửa</h2>
        <input type="text" v-model="inputValueUpdate" />
        <p v-show="errorUpdate" style="color: red">
          Tên công việc không được để trống hoặc trùng
        </p>
        <div class="btn-delete">
          <button @click="closeFormUpdate">Hủy</button>
          <button @click="acceptUpdate">Cập nhật</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import axios from "axios";
import { computed, ref } from "vue";

const jobs = ref([]);
const isLoading = ref(true);
const filter = ref("all");
const getAllJobs = async () => {
  try {
    const res = await axios.get("http://localhost:3000/jobs");
    jobs.value = res.data;
  } catch (error) {
    console.error("Lỗi khi lấy công việc:", error);
  } finally {
    isLoading.value = false;
  }
};
getAllJobs();
const isVisibleDelete = ref(false);
const getJob = ref({});
const onCancel = () => {
  isVisibleDelete.value = false;
};
const handleFormDelete = (job) => {
  getJob.value = job;
  isVisibleDelete.value = true;
};
const deleteTask = async () => {
  await axios.delete(`http://localhost:3000/jobs/${getJob.value.id}`);
  onCancel();
  getAllJobs();
};
const inputValue = ref("");
const errorAdd = ref(false);
const addJob = async () => {
  const findJob = jobs.value.find((item) => item.name === inputValue.value);
  const newJob = {
    name: inputValue.value,
    status: false,
  };
  if (inputValue.value == "") {
    errorAdd.value = true;
  } else if (findJob) {
    errorAdd.value = true;
  } else {
    await axios.post("http://localhost:3000/jobs", newJob);
    errorAdd.value = false;
    inputValue.value = "";
    getAllJobs();
  }
};
const handleCheck = async (job) => {
  job.status = !job.status;
  try {
    await axios.put(`http://localhost:3000/jobs/${job.id}`, job);
  } catch (error) {
    console.error(error);
  }
};
const isVisibleUpdate = ref(false);
const inputValueUpdate = ref("");
const errorUpdate = ref(false);
const getJobUpdate = ref({});
const handleFormEdit = (job) => {
  isVisibleUpdate.value = true;
  inputValueUpdate.value = job.name;
  getJobUpdate.value = job;
};
const closeFormUpdate = () => {
  isVisibleUpdate.value = false;
};
const acceptUpdate = async () => {
  const findJob = jobs.value.find(
    (item) => item.name === inputValueUpdate.value
  );
  const updateJob = {
    name: inputValueUpdate.value,
  };

  if (inputValueUpdate.value == "") {
    errorUpdate.value = true;
  } else if (findJob) {
    errorUpdate.value = true;
  } else {
    await axios.patch(
      `http://localhost:3000/jobs/${getJobUpdate.value.id}`,
      updateJob
    );
    errorUpdate.value = false;
    inputValueUpdate.value = "";
    isVisibleUpdate.value = false;
    getAllJobs();
  }
};
const handleUpdateIncompleteJobs = async () => {
  const completedJobs = jobs.value.filter((item) => item.status === true);
  try {
    await Promise.all(
      completedJobs.map((job) =>
        axios.delete(`http://localhost:3000/jobs/${job.id}`)
      )
    );
    getAllJobs();
  } catch (error) {
    console.error("Lỗi khi xóa công việc hoàn thành:", error);
  }
};
const handleDeleteAll = async () => {
  try {
    await Promise.all(
      jobs.value.map((job) =>
        axios.delete(`http://localhost:3000/jobs/${job.id}`)
      )
    );
    getAllJobs();
  } catch (error) {
    console.error("Lỗi khi xóa tất cả công việc:", error);
  }
};
const filteredJobs = computed(() => {
  if (filter.value === "completed") {
    return jobs.value.filter((job) => job.status);
  } else if (filter.value === "in-progress") {
    return jobs.value.filter((job) => !job.status);
  }
  return jobs.value;
});
const setFilter = (newFilter) => {
  filter.value = newFilter;
};
</script>

<style scoped>
.task-manager {
  max-width: 1000px;
  width: 700px;
  margin: 0 auto;
  padding: 20px;
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

h2 {
  text-align: center;
  margin-bottom: 20px;
}

.add-task {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.add-task input {
  flex: 1;
  padding: 10px;
  border-radius: 5px;
  border: 1px solid #ccc;
}

.add-task-btn {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.task-filters {
  display: flex;
  justify-content: space-around;
  margin-bottom: 20px;
}

.filter {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  background-color: #f1f1f1;
}

.filter.active {
  background-color: #007bff;
  color: white;
}

.loading-spinner {
  text-align: center;
  font-size: 18px;
  color: #007bff;
  margin-bottom: 20px;
}

.task-list {
  list-style: none;
  padding: 0;
  margin: 0 0 20px 0;
  max-height: 220px;
  overflow-y: auto;
}

.task-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px;
  border-bottom: 1px solid #f1f1f1;
}

.task-item input[type="checkbox"] {
  margin-right: 10px;
}

.task-actions {
  display: flex;
  gap: 10px;
}

.edit-task,
.delete-task {
  background: none;
  border: none;
  cursor: pointer;
}

.completed {
  text-decoration: line-through;
  color: #888;
}

.action-buttons {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.clear-completed,
.clear-all {
  padding: 10px 20px;
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}
.modal-content {
  background: white;
  padding: 20px;
  border-radius: 5px;
  text-align: center;
}

.modal .btn-delete {
  display: flex;
  flex-direction: row;
  justify-content: space-around;
}

.modal button {
  padding: 10px 20px;
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
</style>
