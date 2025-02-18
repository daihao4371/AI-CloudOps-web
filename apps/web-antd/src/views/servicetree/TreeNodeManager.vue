<template>
  <div>
    <!-- 操作工具栏 -->
    <div class="custom-toolbar">
      <div class="search-filters">
        <a-input 
          v-model:value="searchText" 
          placeholder="请输入节点名称" 
          style="width: 200px" 
        />
      </div>
      <div class="action-buttons">
        <a-button type="primary" @click="handleAddNode">新增节点</a-button>
      </div>
    </div>

    <!-- 节点列表 -->
    <a-table :columns="columns" :data-source="filteredData" row-key="key">
      <!-- 操作列 -->
      <template #action="{ record }">
        <a-space>
          <a-button type="link" @click="handleEditNode(record)">编辑</a-button>
          <a-button type="link" danger @click="handleDeleteNode(record)"
            >删除</a-button
          >
        </a-space>
        <!-- 编辑表单的模态框 -->
        <a-modal
          v-model:visible="isEditModalVisible"
          title="编辑节点"
          @ok="handleSaveNode"
          @cancel="handleCancel"
        >
          <a-form
            :model="editForm"
            :label-col="{ span: 6 }"
            :wrapper-col="{ span: 16 }"
            ref="editFormRef"
          >
            <a-form-item
              label="节点名称"
              name="title"
              :rules="[{ required: true, message: '请输入节点名称' }]"
            >
              <a-input
                v-model:value="editForm.title"
                placeholder="请输入节点名称"
              />
            </a-form-item>

            <a-form-item label="描述" name="description">
              <a-input v-model:value="editForm.desc" placeholder="请输入描述" />
            </a-form-item>

            <a-form-item label="运维负责人" name="ops_admins">
              <a-select
                v-model:value="editForm.ops_admins"
                mode="multiple"
                placeholder="请选择运维负责人"
              >
                <a-select-option
                  v-for="person in availableOpsAdmins"
                  :key="person.id"
                  :value="person.id"
                >
                  {{ person.name }}
                </a-select-option>
              </a-select>
            </a-form-item>

            <a-form-item label="研发负责人" name="rd_admins">
              <a-select
                v-model:value="editForm.rd_admins"
                mode="multiple"
                placeholder="请选择研发负责人"
              >
                <a-select-option
                  v-for="person in availableRdAdmins"
                  :key="person.id"
                  :value="person.id"
                >
                  {{ person.name }}
                </a-select-option>
              </a-select>
            </a-form-item>

            <a-form-item label="研发工程师" name="rd_members">
              <a-select
                v-model:value="editForm.rd_members"
                mode="multiple"
                placeholder="请选择研发工程师"
              >
                <a-select-option
                  v-for="person in availableRdMembers"
                  :key="person.id"
                  :value="person.id"
                >
                  {{ person.name }}
                </a-select-option>
              </a-select>
            </a-form-item>
          </a-form>
        </a-modal>
      </template>
    </a-table>

    <!-- 新增节点的模态框 -->
    <a-modal
      v-model:visible="isAddModalVisible"
      title="新增节点"
      @ok="handleSaveAddNode"
      @cancel="isAddModalVisible = false"
    >
      <a-form
        :model="addForm"
        :label-col="{ span: 6 }"
        :wrapper-col="{ span: 16 }"
      >
        <a-form-item
          label="节点名称"
          name="title"
          :rules="[{ required: true, message: '请输入节点名称' }]"
        >
          <a-input v-model:value="addForm.title" placeholder="请输入节点名称" />
        </a-form-item>

        <a-form-item label="描述" name="desc">
          <a-input v-model:value="addForm.desc" placeholder="请输入描述" />
        </a-form-item>

        <a-form-item label="父节点" name="pId">
          <a-tree-select
            v-model:value="addForm.pId"
            :tree-data="data"
            :field-names="{
              children: 'children',
              label: 'title',
              value: 'id',
            }"
            placeholder="请选择父节点"
            :tree-default-expand-all="true"
            :show-search="true"
            :filter-tree-node="filterTreeNode"
          />
        </a-form-item>

        <a-form-item label="层级" name="level">
          <a-input-number v-model:value="addForm.level" :min="1" />
        </a-form-item>

        <a-form-item label="节点类型" name="isLeaf">
          <a-select v-model:value="addForm.isLeaf" placeholder="请选择节点类型">
            <a-select-option :value="0">目录节点</a-select-option>
            <a-select-option :value="1">叶子节点</a-select-option>
          </a-select>
        </a-form-item>
      </a-form>
    </a-modal>
  </div>
</template>

<script lang="ts" setup>
import { reactive, ref, onMounted, watch } from 'vue';
import { message, Modal } from 'ant-design-vue';
import { getAllTreeNodes, deleteTreeNode, updateTreeNode, createTreeNode } from '#/api';
import type { TreeNode, User } from '#/api/core/tree';
// 节点数据
const data = reactive<TreeNode[]>([]);
const isEditModalVisible = ref(false);
// 搜索文本
const searchText = ref('');
// 过滤后的数据
const filteredData = ref<TreeNode[]>(data);

// 监听搜索文本变化
watch(searchText, (newValue) => {
  const searchValue = newValue.trim().toLowerCase();
  filteredData.value = data.filter((item) =>
    item.title.toLowerCase().includes(searchValue),
  );
});

// 表格列配置
const columns = [
  {
    title: '节点名称',
    dataIndex: 'title',
    key: 'title',
  },
  {
    title: 'id',
    dataIndex: 'id',
    key: 'id',
  },
  {
    title: '层级',
    dataIndex: 'level',
    key: 'level',
  },
  {
    title: '描述',
    dataIndex: 'desc',
    key: 'desc',
  },
  {
    title: '是否为叶子节点',
    dataIndex: 'isLeaf',
    key: 'isLeaf',
    customRender: ({ isLeaf }: { isLeaf: number }) => (isLeaf ? 1 : 0),
  },
  {
    title: 'ECS 数量',
    dataIndex: 'ecsCount',
    key: 'ecsCount',
  },
  {
    title: 'ELB 数量',
    dataIndex: 'elbCount',
    key: 'elbCount',
  },
  {
    title: 'RDS 数量',
    dataIndex: 'rdsCount',
    key: 'rdsCount',
  },
  {
    title: '操作',
    key: 'action',
    slots: { customRender: 'action' },
  },
];

// 编辑表单的数据
const editForm = reactive({
  id: 0,
  title: '',
  desc: '',
  ops_admins: [] as User[],
  rd_admins: [] as User[],
  rd_members: [] as User[],
});

// 可选的运维负责人列表 假数据
const availableOpsAdmins = [
  { id: 1, name: '运维负责人A' },
  { id: 2, name: '运维负责人B' },
];

// 可选的研发负责人列表 假数据
const availableRdAdmins = [
  { id: 3, name: '研发负责人A' },
  { id: 4, name: '研发负责人B' },
];

// 可选的研发工程师列表 假数据
const availableRdMembers = [
  { id: 5, name: '研发工程师A' },
  { id: 6, name: '研发工程师B' },
];

const handleDeleteNode = (record: TreeNode) => {
  // 手动创建一个确认 Modal
  Modal.confirm({
    title: '确认删除',
    content: `确认删除节点 "${record.title}" 吗?`,
    okText: '确认',
    cancelText: '取消',
    onOk: async () => {
      try {
        // 调用后端删除接口，传入节点的 key 或 id
        await deleteTreeNode(record.id); // 假设 key 是节点的唯一标识

        // 从前端数据中删除节点
        const index = data.findIndex((item) => item.key === record.key);
        if (index !== -1) {
          data.splice(index, 1);
          message.success(`节点 "${record.title}" 已删除`);
        }
      } catch (err) {
        // 捕获删除失败的错误并展示错误信息
        message.error(String(err.message));
      }
    },
    onCancel() {
      console.log('取消删除');
    },
  });
};

const isAddModalVisible = ref(false);

// 添加节点的表单数据
const addForm = reactive({
  title: '',
  desc: '',
  pId: 0,
  isLeaf: 0,
  level: 1,
});

const handleAddNode = () => {
  // 重置表单数据
  Object.assign(addForm, {
    title: '',
    desc: '',
    pId: 0,
    isLeaf: 0,
    level: 1,
  });
  
  // 显示添加节点的模态框
  isAddModalVisible.value = true;
};

// 添加保存新节点的方法
const handleSaveAddNode = async () => {
  if (!addForm.title) {
    message.error('节点名称不能为空');
    return;
  }

  try {
    await createTreeNode({
      title: addForm.title,
      pId: addForm.pId,
      desc: addForm.desc,
      isLeaf: addForm.isLeaf,
      level: addForm.level,
    });

    message.success('新增节点成功');
    isAddModalVisible.value = false;
    await refreshTreeData(); // 刷新节点数据
  } catch (error) {
    message.error('新增节点失败');
    console.error(error);
  }
};

// 点击编辑按钮时，弹出表单并填充默认数据
const handleEditNode = (record: TreeNode) => {
  // 填充编辑表单的数据
  editForm.id = record.id;
  editForm.title = record.title;
  editForm.desc = record.desc;
  editForm.ops_admins = [...record.ops_admins]; // 复制运维负责人
  editForm.rd_admins = [...record.rd_admins]; // 复制研发负责人
  editForm.rd_members = [...record.rd_members]; // 复制研发工程师

  // 显示模态框
  isEditModalVisible.value = true;
};

// 保存节点数据
const handleSaveNode = async () => {
  try {
    // 等待更新节点的请求完成
    await updateTreeNode(editForm);

    // 更新成功后显示成功消息
    message.success('节点信息已保存');

    // 关闭模态框
    isEditModalVisible.value = false;

    // 刷新页面上的节点数据，确保页面数据是最新的
    await refreshTreeData(); // 假设这个函数用于重新获取最新的节点数据并更新页面
  } catch (error) {
    // 如果更新失败，显示错误消息
    message.error('保存节点信息失败');
    console.error(error);
  }
};

// 取消编辑
const handleCancel = () => {
  isEditModalVisible.value = false;
};

// 树选择器的搜索过滤函数
const filterTreeNode = (inputValue: string, treeNode: any) => {
  return treeNode.title.toLowerCase().indexOf(inputValue.toLowerCase()) >= 0;
};

onMounted(() => {
  getAllTreeNodes()
    .then((response) => {
      if (response) {
        data.splice(0, data.length, ...response); // 替换 reactive 对象中的数据
        filteredData.value = data; // 初始化时，将 filteredData 设置为 data
      } else {
        // 如果后端返回空数据,将数据设为空数组
        data.splice(0, data.length);
        filteredData.value = [];
      }
    })
    .catch((error) => {
      message.error('获取树数据失败');
      console.error(error);
    });
});

// 获取所有节点数据并更新页面
const refreshTreeData = async () => {
  try {
    const response = await getAllTreeNodes(); // 调用 API 获取所有节点数据
    if (!response) {
      // 如果后端返回空数据,将数据设为空数组
      data.splice(0, data.length);
      return;
    }
    data.splice(0, data.length, ...response); // 更新页面显示的数据
  } catch (error) {
    message.error('刷新树节点数据失败');
    console.error(error);
  }
};
</script>

<style scoped>
.custom-toolbar {
  padding: 8px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.search-filters {
  display: flex;
  align-items: center;
}

.action-buttons {
  display: flex;
  align-items: center;
}
</style>
