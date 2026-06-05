---
date: '2026-06-05T16:00:56+08:00'
draft: false
title: 'vue中的表单校验'
tags: ["Vue"]
categories: ["big-event"]
---

以下是一个表单的例子：我想说的是如果不加el-form中的ref,那么如果直接点击“提交”按钮，那么就会把错误的数据提交给后端。这是不对的
```vue
<script lang="ts" setup>
const form = ref({
    oldPassword: '',
    newPassword: '',
    rePassword: ''
})
const formRef = ref()
const onSubmit = async () => {
    await formRef.value.validate()
    console.log(form.value)
    await updatePasswordService(form.value)
    ElMessage.success('密码更新成功')
    router.push('/login')
}
</script>

<template>
    <el-form ref="formRef" :model="form" label-width="auto" style="max-width: 600px" :rules="rules">
        <el-form-item label="旧密码" prop="oldPassword">
            <el-input v-model="form.oldPassword" type="password" />
        </el-form-item>
        <el-form-item label="新密码" prop="newPassword">
            <el-input v-model="form.newPassword" type="password" />
        </el-form-item>
        <el-form-item label="重复新密码" prop="rePassword">
            <el-input v-model="form.rePassword" type="password" />
        </el-form-item>
        <el-form-item label="    ">
            <el-button type="primary" @click="onSubmit">确认提交</el-button>
        </el-form-item>
    </el-form>
</template>
```

如代码所示，需要给表单el-form一个ref="formRef"属性，指向一个空的ref变量。然后在submit方法中  await formRef.value.validate()

这时如果什么都不做，上来点击"提交"就会触发验证操作。

顺带说一句：

如果obSubmit方法中的 await updatePasswordService(form.value)发生错误，比如返回失败等，会直接返回Promise.error，是因为我配置了响应拦截器：

```js
// 添加请求拦截器
instance.interceptors.request.use(
    config=>{
        const tokenStore = useTokenStore()
        if(tokenStore.token){
            config.headers.Authorization = tokenStore.token
        }
        return config;
    },
    err => {
        return Promise.reject(err);
    }
)

//添加响应拦截
instance.interceptors.response.use(
    result=>{
        if(result.data.code === 0){
            return result.data;
        }else{
            ElMessage.error(result.data.message? result.data.message : '服务异常');
            return Promise.reject(result.data);
        }
    },
    err=>{
        if(err.response.status === 401){
            ElMessage.error('请先登录');
            router.push('/login');
        }
        else{
            ElMessage.error('服务异常');
        }
        return Promise.reject(err);//异步的状态转化成失败的状态
    }
)
```

可以看到在响应拦截中，如果返回的code不是0，那么就会弹错误信息，返回Promise.reject。

再回到onSubmit方法中，await返回的reject就如同throw一样，发生错误，后续的代码都不执行。于是 ElMessage.success('密码更新成功')和router.push('/login')都不会执行。