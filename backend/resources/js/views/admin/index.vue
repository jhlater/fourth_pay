<template>
    <div class="app-container" v-loading="loading">
        <div class="container">
            <div class="handle-box">
                <!--
                <el-button type="primary" icon="el-icon-delete" class="handle-del mr10" @click="delAll" size="small">批量删除</el-button>
                <el-select v-model="select_cate" placeholder="筛选省份" class="handle-select mr10" size="small">
                    <el-option key="1" label="广东省" value="广东省" ></el-option>
                    <el-option key="2" label="湖南省" value="湖南省" ></el-option>
                </el-select>
                -->
                <el-input v-model="seach.dminname" placeholder="筛选关键词" class="handle-input mr10" size="small"></el-input>
                <el-button type="primary" icon="el-icon-search" @click="handleSearch" size="small">搜索</el-button>
                <el-button type="primary" icon="el-icon-circle-plus-outline" v-permission="'admin/create'" @click="handleAddAdmin" size="small">新增管理员</el-button>
            </div>

            <el-table :data="adminList" style="width: 100%;margin-top:30px;" border>
                <el-table-column align="center" label="ID" prop="id"></el-table-column>
                <el-table-column align="center" label="用户名" prop="username"></el-table-column>
                <el-table-column align="header-center" label="昵称" prop="nickname"></el-table-column>
                <el-table-column align="header-center" label="是否锁定" >
                    <template slot-scope="scope">
                        <el-tag type="success" v-if="!scope.row.is_locked">正常</el-tag>
                        <el-tag type="danger" v-else>锁定</el-tag>
                    </template>
                </el-table-column>
                <el-table-column align="header-center" label="上次登录IP" prop="last_ip"></el-table-column>
                <el-table-column align="header-center" label="上次登录时间" prop="last_time"></el-table-column>
                <el-table-column align="center" label="Operations">
                    <template slot-scope="scope" v-if="scope.row.id!=1">
                        <el-button type="primary" size="small" @click="handleEdit(scope)" v-permission="'admin/edit'">Edit</el-button>
                        <el-button type="danger" size="small" @click="handleDelete(scope)" v-permission="'admin/delete'">Delete</el-button>
                    </template>
                </el-table-column>
            </el-table>

            <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getAdminUsers" />
        </div>

        <el-dialog :visible.sync="dialogVisible" width="580px" :title="dialogType==='edit'?'Edit Admin':'New Admin'">
            <el-form :model="admin" label-width="15%" label-position="right">
                <el-form-item label="用户名">
                    <el-input v-model="admin.username" placeholder="用户名" />
                </el-form-item>
                <el-form-item label="昵称">
                    <el-input v-model="admin.nickname" placeholder="昵称" />
                </el-form-item>
                <el-form-item label="密码">
                    <el-input v-model="admin.password" placeholder="密码" show-password/>
                </el-form-item>
                <el-form-item label="锁定">
                    <el-switch v-model="admin.is_locked" ></el-switch>
                </el-form-item>
                <el-form-item label="角色">
                    <el-checkbox-group v-model="admin.role" >
                        <el-checkbox :label="item.id" v-for="(item,key) in rolesList" :key="key">{{item.name}}</el-checkbox>
                    </el-checkbox-group>
                </el-form-item>
            </el-form>
            <div style="text-align:right;">
                <el-button type="danger" @click="dialogVisible=false">Cancel</el-button>
                <el-button type="primary" @click="confirm">Confirm</el-button>
            </div>
        </el-dialog>
    </div>
</template>

<script>
    import permission from '@/directive/permission/index.js' // 权限判断指令
    import { getAllAdmins,addAdmin,editAdmin,getAdminUser,deleteAdmin } from '@/api/admin'
    import { getAllRoles } from '@/api/role'
    import Pagination from '@/components/Pagination' // Secondary package based on el-pagination

    const defaultAdmin = {
        id : '',
        username : '',
        nickname : '',
        password : '',
        is_locked : false,
        role : [],
    };

    export default{
        data(){
            return {
                admin: Object.assign({}, defaultAdmin),
                adminList: [],
                rolesList: [],
                dialogVisible: false,
                dialogType: 'new',
                loading:false,
                total: 0,
                listQuery: {
                    page: 1,
                    limit: 20
                },
                seach:{
                    adminname:'',
                }
            };
        },
        components: { Pagination },
        computed: {
        },
        directives: { permission },
        created() {
            this.getAdminUsers();
            this.getRoles();
        },
        methods: {
            async getAdminUsers(){
                this.loading =  true;
                let result = await getAllAdmins(this.listQuery);
                if( result.data.code == 1 ){
                    this.total = result.data.data.total;
                    this.adminList = result.data.data.adminlist;
                }else{
                    this.$message.error(result.data.msg);
                }
                this.loading =  false;
            },
            async getRoles() {
                let result = await getAllRoles()
                if( result.data.code == 1 ){
                    this.rolesList = result.data.data.roles
                }else{
                    this.$message.error(result.data.msg);
                }
            },
            handleAddAdmin(){
                this.admin = Object.assign({}, defaultAdmin)
                this.dialogType = 'new'
                this.dialogVisible = true
            },
            async handleEdit( scope ){
                this.loading =  true;
                let current_user = await getAdminUser(scope.row.id);
                this.admin = current_user.data.data;
                this.dialogType = 'edit'
                this.dialogVisible = true
                this.loading =  false;
            },
            handleDelete(scope ){
                this.$confirm('此操作将永久删除该管理员, 是否继续?', '提示', {
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    type: 'warning'
                }).then( async() => {
                    let result = await deleteAdmin( scope.row.id )
                    if( result.data.code == 1 ){
                        this.$message({
                            type: 'success',
                            message: '删除成功!'
                        });
                        this.getAdminUsers();
                    }else{
                        this.$message.error(result.data.msg);
                    }
                }).catch((e) => {
                    console.log(e);
                });
            },
            async confirm(){
                const isEdit = this.dialogType === 'edit'

                let type = 'error';
                let message = '';

                let response;

                if (isEdit) {
                    response = await editAdmin(this.admin)
                }else{
                    response = await addAdmin(this.admin)
                }

                if( response.data.code == 1 ){
                    type = 'success';
                    message = `
                            <div>Admin name: ${this.admin.name}</div>
                          `;
                }else{
                    message = response.data.msg;
                }

                this.dialogVisible = false

                this.getAdminUsers();

                this.$notify({
                    title: response.data.msg,
                    dangerouslyUseHTMLString: true,
                    message: message,
                    type: type
                })
            },
            /**
             * 查找管理员
             */
            handleSearch(){
                this.getAdminUsers();
            }
        }
    }
</script>