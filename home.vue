<template>
  <div id="app">
    <el-container style="height: 100vh;">
      <!-- 页眉部分 -->
      <el-header height="60px" class="header">
        <div class="header-title">
          <i class="el-icon-phone-outline"></i>
          通讯录
        </div>
      </el-header>

      <el-container>
        <el-container>
          <!-- 搜索栏部分 -->
          <el-header height="auto" class="search-header">
            <el-row :gutter="60" class="search-row" style="justify-content: center;">
              <el-col :span="12">
                <!-- 搜索框 -->
                <el-input v-model="searchQuery" placeholder="请输入姓名、电话、邮箱或地址进行搜索" prefix-icon="el-icon-search"
                  @input="filterContacts" />
              </el-col>
              <el-col :span="12" class="header-buttons">
                <!-- 添加联系人按钮 -->
                <el-button class="add-contact-button" type="primary" icon="el-icon-plus"
                  @click="showAddDialog">添加联系人</el-button>
                <!-- 批量删除按钮 -->
                <el-button class="delete-contacts-button" type="danger" icon="el-icon-delete"
                  @click="deleteSelectedContacts" :disabled="!selectedContacts.length">批量删除</el-button>
                <!-- 显示收藏按钮 -->
                <el-button class="show-favorites-button" type="info" @click="toggleShowFavorites">
                  {{ showFavorites ? '显示所有联系人' : '显示收藏联系人' }}
                </el-button>
                <!-- 导出按钮 -->
                <el-button class="export-contacts-button" type="success" icon="el-icon-download" @click="exportContacts">
                  导出所有联系人
                </el-button>
                <!-- 导入按钮 -->
                <el-button class="import-contacts-button" type="info" icon="el-icon-upload" @click="triggerFileInput">
                  导入联系人
                </el-button>
                <input type="file" ref="fileInput" style="display: none;" @change="importContacts" />
              </el-col>
            </el-row>
          </el-header>

          <el-main style="padding: 20px; display: flex; justify-content: center;">
            <div style="width: 1156px;">
              <!-- 联系人表格 -->
              <el-table :data="paginatedContacts" border stripe style="width: 100%;" :height="tableHeight"
                @selection-change="handleSelectionChange">
                <!-- 选择框列 -->
                <el-table-column type="selection" width="55"></el-table-column>
                <!-- 姓名列 -->
                <el-table-column prop="name" label="姓名" width="150"></el-table-column>
                <!-- 电话列 -->
                <el-table-column prop="phone1" label="电话1" width="180"></el-table-column>
                 <!-- 电话列 -->
                 <el-table-column prop="phone2" label="电话2" width="180"></el-table-column>
                <!-- 邮箱列 -->
                <el-table-column prop="email" label="邮箱" width="220"></el-table-column>
                <!-- 地址列 -->
                <el-table-column prop="address" label="地址" width="250"></el-table-column>
                <!-- 是否收藏列 -->
                <el-table-column label="是否收藏" width="120">
                  <template #default="scope">
                    <el-switch v-model="scope.row.isFavorite" @change="toggleFavorite(scope.row)" active-text="是" inactive-text="否"></el-switch>
                  </template>
                </el-table-column>
                <!-- 操作列，包括编辑和删除按钮 -->
                <el-table-column label="操作" width="180">
                  <template #default="scope">
                    <el-button class="edit-button" size="small" icon="el-icon-edit"
                      @click="editContact(scope.row)">编辑</el-button>
                    <el-button class="delete-button" size="small" icon="el-icon-delete"
                      @click="deleteContact(scope.row.id)">删除</el-button>
                  </template>
                </el-table-column>
              </el-table>
            </div>

            <!-- 分页组件 -->
            <el-pagination background layout="prev, pager, next" :total="filteredContacts.length" :page-size="pageSize"
              :current-page.sync="currentPage" @current-change="handlePageChange" class="pagination"
              style="justify-content: center;" />
          </el-main>
        </el-container>
      </el-container>
    </el-container>

    <!-- 添加/编辑联系人对话框 -->
    <el-dialog :visible.sync="dialogVisible" title="联系人信息" width="500px" class="contact-dialog" :draggable="disable">
      <el-form :model="contactForm" label-width="80px" class="contact-form"
        style="margin: 0 auto; display: flex; flex-direction: column; align-items: center;">
        <!-- 姓名输入框 -->
        <el-form-item label="姓名">
          <el-input v-model="contactForm.name"></el-input>
        </el-form-item>
        <!-- 电话输入框 -->
        <el-form-item label="电话1">
          <el-input v-model="contactForm.phone1"></el-input>
        </el-form-item>
        <el-form-item label="电话2">
          <el-input v-model="contactForm.phone2"></el-input>
        </el-form-item>
        <!-- 邮箱输入框 -->
        <el-form-item label="邮箱">
          <el-input v-model="contactForm.email"></el-input>
        </el-form-item>
        <!-- 地址输入框 -->
        <el-form-item label="地址">
          <el-input v-model="contactForm.address"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer" style="text-align: center;">
        <!-- 取消按钮 -->
        <el-button @click="dialogVisible = false">取消</el-button>
        <!-- 确定按钮 -->
        <el-button type="primary" @click="submitForm">确定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import request from '../utils/request';
import * as XLSX from 'xlsx';

export default {
  data() {
    return {
      contacts: [],
      filteredContacts: [],
      searchQuery: '',
      dialogVisible: false,
      contactForm: {
        id: null,
        name: '',
        phone1: '',
        phone2: '',
        email: '',
        address: ''
      },
      currentPage: 1,
      pageSize: 12,
      isEditing: false,
      tableHeight: 0,
      selectedContacts: [],
      showFavorites: false
    };
  },
  created() {
    this.fetchContacts();
    this.updateTableHeight();
    window.addEventListener('resize', this.updateTableHeight);
  },
  computed: {
    paginatedContacts() {
      const start = (this.currentPage - 1) * this.pageSize;
      const end = this.currentPage * this.pageSize;
      return this.filteredContacts.slice(start, end);
    }
  },
  methods: {
    fetchContacts() {
      request.get('/user/selectAll')
        .then(response => {
          this.contacts = response.data;
          this.filterContacts();
        })
        .catch(error => {
          console.error(error);
        });
    },
    showAddDialog() {
      this.isEditing = false;
      this.contactForm = { id: null, name: '', phone: '', email: '', address: '' };
      this.dialogVisible = true;
    },
    editContact(contact) {
      this.isEditing = true;
      this.contactForm = { ...contact };
      this.dialogVisible = true;
    },
    submitForm() {
      if (!this.contactForm.name) {
        this.$message.error('姓名不能为空');
        return;
      }
      if (this.isEditing) {
        this.updateContact();
      } else {
        this.addContact();
      }
    },
    addContact() {
      request.post('/user/add', this.contactForm)
        .then(response => {
          this.dialogVisible = false;
          this.fetchContacts();
          this.$message({ message: '添加联系人成功', type: 'success' });
        })
        .catch(error => {
          console.error('Error adding contact:', error);
        });
      this.currentPage = 1;
    },
    updateContact() {
      request.put(`/user/update/${this.contactForm.id}`, this.contactForm)
        .then(() => {
          this.dialogVisible = false;
          this.fetchContacts();
          this.$message({ message: '更新联系人成功', type: 'success' });
        })
        .catch(error => {
          console.error('Error updating contact:', error);
        });
    },
    deleteContact(contactId) {
      request.delete(`/user/delete/${contactId}`)
        .then(() => {
          this.fetchContacts();
          this.$message({ message: '删除联系人成功', type: 'success' });
        })
        .catch(error => {
          console.error(error);
        });
    },
    deleteSelectedContacts() {
      const contactIds = this.selectedContacts.map(contact => contact.id);
      request.delete('/user/deleteBatch', { data: { ids: contactIds } })
        .then(() => {
          this.fetchContacts();
          this.$message({ message: '批量删除联系人成功', type: 'success' });
        })
        .catch(error => {
          console.error(error);
        });
    },
    handlePageChange(page) {
      this.currentPage = page;
    },
    updateTableHeight() {
      const headerHeight = 60;
      const footerHeight = 50;
      this.tableHeight = window.innerHeight - headerHeight - footerHeight - 60;
    },
    handleSelectionChange(val) {
      this.selectedContacts = val;
    },
    filterContacts() {
      const query = this.searchQuery.toLowerCase();
      this.filteredContacts = this.contacts.filter(contact => {
        return contact.name.toLowerCase().includes(query) ||
          contact.phone.toLowerCase().includes(query) ||
          contact.email.toLowerCase().includes(query) ||
          contact.address.toLowerCase().includes(query);
      });
      if (this.showFavorites) {
        this.filteredContacts = this.filteredContacts.filter(contact => contact.isFavorite);
      }
    },
    toggleFavorite(contact) {
      request.put(`/user/update/${contact.id}`, { isFavorite: contact.isFavorite })
        .then(() => {
          this.$message({ message: '收藏状态已更新', type: 'success' });
        })
        .catch(error => {
          console.error(error);
        });
    },
    toggleShowFavorites() {
      this.showFavorites = !this.showFavorites;
      this.filterContacts();
    },
    exportContacts() {
      const worksheet = XLSX.utils.json_to_sheet(this.filteredContacts);
      const workbook = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(workbook, worksheet, 'Contacts');
      XLSX.writeFile(workbook, 'contacts.xlsx');
    },
    importContacts(event) {
      const file = event.target.files[0];
      const reader = new FileReader();
      reader.onload = (e) => {
        const data = e.target.result;
        const workbook = XLSX.read(data, { type: 'binary' });
        const sheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[sheetName];
        const contacts = XLSX.utils.sheet_to_json(worksheet);
        this.addImportedContacts(contacts);
      };
      reader.readAsBinaryString(file);
    },
    addImportedContacts(contacts) {
      contacts.forEach(contact => {
        request.post('/user/add', contact)
          .then(() => {
            this.fetchContacts();
          })
          .catch(error => {
            console.error('导入联系人失败:', error);
          });
      });
    },
    triggerFileInput() {
      this.$refs.fileInput.click();
    }
  }
};
</script>

<style scoped>
/* 样式部分 */
.header {
  background-color: #409EFF;
  color: #fff;
  display: flex;
  align-items: center;
  padding: 0 20px;
}

.header-title {
  font-size: 24px;
  font-weight: bold;
}

.search-header {
  background-color: #f0f2f5;
  padding: 10px;
}

.search-row {
  display: flex;
  justify-content: space-between;
}

.header-buttons {
  display: flex;
  align-items: center;
}

.export-contacts-button,
.import-contacts-button {
  margin-left: 10px;
}

.contact-dialog {
  padding: 20px;
}

.dialog-footer {
  padding: 10px 0;
  display: flex;
  justify-content: space-between;
}

.pagination {
  margin-top: 20px;
}

.delete-button {
  margin-left: 10px;
}

.edit-button {
  margin-left: 10px;
}

.delete-contacts-button {
  margin-left: 10px;
}
</style>
