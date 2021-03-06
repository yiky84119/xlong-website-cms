<template>
    <div class="g-content">
        <!--  加载判断 -->
        <Loading v-if="pageLoading"></Loading>
        <div v-else>
            <Form ref="infoForm" :model="infoForm" :rules="validate" :label-width="100">
                <Form-item label="文章标题：" prop="title">
                    <Input v-model="infoForm.title" placeholder="请输入文章标题"/>
                </Form-item>
                <Form-item label="文章封面：" prop="imgUrl">
                    <!-- 组件-图片上传-图片裁剪 -->
                    <ImageCropper :src="infoForm.imgUrl" upload-dir="img/article/" :preview="true" :fixed="true" :fixedNumber="[90,59]" :autoCropWidth="450" :fileSize="300" size-hint="450*295px" @get-img-url="setPoster"></ImageCropper>
                </Form-item>
                <Form-item label="作者：">
                    <Input v-model="infoForm.author" placeholder="请输入作者名称"/>
                </Form-item>
                <Form-item label="文章摘要：">
                    <Input v-model="infoForm.summary" type="textarea" :rows="3" :maxlength="50" placeholder="请输入文章摘要(不超过50个字符)"/>
                </Form-item>
                <Form-item label="关键词：">
                    <Tag v-for="(item, index) in infoForm.metaKeywords" :key="index" :name="item" closable @on-close="removeKeyword(index)">{{ item }}</Tag>
                    <Button icon="ios-add" type="dashed" size="small" @click="kwModelVisible(true)">添加关键词</Button>
                </Form-item>
                <Form-item label="页面描述：">
                    <Input v-model="infoForm.metaDescription" type="textarea" :rows="3" :maxlength="200" placeholder="请输入文章描述（不超过200个字符）"/>
                </Form-item>
                <Form-item label="文章内容：" prop="content">
                    <!-- 组件-富文本编辑器 -->
                    <Tinymce v-model="infoForm.content" placeholder="请输入文章内容" height="500" />
                </Form-item>
                <!-- 操作按钮 -->
                <div class="m-operation">
                    <Button v-if="pageType == 'edit'" class="fr" type="primary" @click="submit('infoForm')">确认修改</Button>
                    <Button v-else class="fr" type="primary" @click="submit('infoForm')">确认新增</Button>
                    <Button class="fr" type="default" @click="$router.go(-1)">返回</Button>
                    <div class="clearfix"></div>
                </div>
            </Form>
        </div>
        <!-- 添加关键词 -->
        <Modal v-model="kwModelShow" :width="600" title="添加关键词" @on-cancel="kwModelVisible(false)">
            <div>
                <Form ref="kwModelForm" :model="kwModelForm" :rules="validate" :label-width="80">
                    <Form-item label="关键词：" prop="keyword">
                        <Input v-model="kwModelForm.keyword" type="textarea" placeholder="请输入关键词"/>
                    </Form-item>

                    <div class="hint">* 输入的关键词请以逗号" , "进行分隔</div>
                </Form>
            </div>
            <div slot="footer">
                <Button type="text" @click="kwModelVisible(false)">取消</Button>
                <Button type="primary" @click="addKeywords('kwModelForm')">确认</Button>
            </div>
        </Modal>
    </div>
</template>

<script>
    // 组件
    import Loading from 'components/Common/Loading'
    import ImageCropper from 'components/Image/ImageCropper'
	import Tinymce from 'components/Common/Tinymce'
    // 通用JS
    import { GetUrlQuery, StrToArr, ArrToStr } from 'utils'
    // 关键词设置
    import KeywordModel from 'mixins/keyword_model.js'
    // Api方法
    import Api from 'api/article_manage.js'

    export default {
        components: { Loading, ImageCropper, Tinymce },
        mixins: [  KeywordModel ],
        data() {
            return {
                // 加载页面
                pageLoading: true,
                // 页面描述
                pageType: 'add',
                // 文章id
                articleId: '',
                // 表单信息
                infoForm: {
                    // 封面地址
                    imgUrl: '',
                    // 标题
                    title: '',
                    // 作者
                    author: '',
                    // 关键词
                    metaKeywords: [],
                    // 页面描述
                    metaDescription: '',
                    // 摘要
                    summary: '',
                    // 内容
                    content: '',
                },
                // 验证规则
                validate: {
                    title: [{ required: true, message: '文章标题不能为空', trigger: 'blur'}],
                    content: [{ required: true, message: '文章内容不能为空', trigger: 'blur'}],
                    imgUrl: [{ required: true, validator: (rule, value, callback) => callback() }],
                    keyword: [{ required: true, message: '关键词不能为空', trigger: 'blur'}]
                }
            }
        },
        created() {
            let id = GetUrlQuery('id');
            // 判断是否是编辑页
            if(id){
                // 获取用户编号
                this.articleId = id;
                this.$store.commit('common/SET_BREADCRUMB', [
                    { name: '新闻资讯', path: '/admin/ArticleManage/List' },
                    { name: '文章详情', path: '' }
                ]);
                // 获取文章详情
                this.getDetail();
                this.pageType = 'edit';
            }
            else{
                this.pageLoading = false;
                this.$store.commit('common/SET_BREADCRUMB', [
                    { name: '新闻资讯', path: '/admin/ArticleManage/List' },
                    { name: '新增文章', path: '' }
                ]);
            }
        },
        methods: {
            // 保存文章
            submit(form) {
                this.$refs[form].validate((valid) => {
                    if (valid) {
                        if(this.infoForm.imgUrl == ''){
                            this.$Message.error('请上传文章封面');
                            return false;
                        }
                        // 页面加载
                        this.pageLoading = true;

                        this.infoForm.metaKeywords = ArrToStr(this.infoForm.metaKeywords, ',');

                        if(this.pageType == 'add'){
                            // 新增
                            Api.AddArticle(this.infoForm)
                            .then(res => {
                                // 取消页面加载
                                this.pageLoading = false;
                                if(res.code == 200){
                                    this.$Message.success({
                                        content: '新增成功',
                                        onClose: () => {
                                            // 跳转到列表页
                                            this.$router.push({ name: 'ArticleManage' });
                                        }
                                    });
                                }
                                else this.$Message.warning(res.msg);
                            })
                            .catch(err => this.$Message.error(err.message))
                        }
                        else{
                            // 修改
                            Api.EditArticle(this.infoForm, this.articleId)
                            .then(res => {
                                // 取消页面加载
                                this.pageLoading = false;
                                if(res.code == 200){
                                    this.$Message.success('修改成功');
                                    this.getDetail();
                                }
                                else this.$Message.warning(res.msg);
                            })
                            .catch(err => console.log(err))
                        }
                    }
                    else this.$Message.error('提交失败！填写有误');
                })
            },
            // 获取文章详情
            getDetail(){
                Api.GetDetail(this.articleId)
                .then(res => {
                    // 取消页面加载
                    this.pageLoading = false;
                    if(res.code == 200){
                        // 设置数据
                        this.infoForm = res.data;
                        this.infoForm.metaKeywords = res.data.metaKeywords ? StrToArr(res.data.metaKeywords, ',') : [];
                    }
                    else this.$Message.warning(res.msg);
                })
                .catch(err => console.log(err))
            },
            // 获取并设置封面
            setPoster(url){
                this.infoForm.imgUrl = url;
            }
        }
    }
</script>

<style lang="less" scoped>
    .m-operation {
        button, a {
            min-width: 80px;
            margin-left: 10px;
        }
    }

    .hint{
        margin-top: 20px;
    }
</style>
