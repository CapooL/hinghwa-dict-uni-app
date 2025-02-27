<template>
  <view>
    <cu-custom title="语记·文章" />

    <scroll-view
      scroll-y
      style="height: 100%"
    >
      <view
        class="padding-sm"
        style="margin-bottom: 120upx;"
      >
        <!--文章标题-->
        <view class="text-bold text-xxl line">
          {{ article.title }}
        </view>

        <!--文章信息-->
        <view>
          <view
            class="flex text-df margin-top align-center"
            @tap="toUserPage(article.author.id)"
          >
            <text class="text-grey">
              文章作者：
            </text>
            <image
              :src="article.author.avatar"
              class="cu-avatar round ssm"
              mode="aspectFill"
            />
            <text class="text-grey">
              {{ article.author.nickname }}
            </text>
          </view>
          <text class="text-grey">
            发布时间：{{ article.publish_time + "\n" }}
          </text>
          <text class="text-grey">
            更新时间：{{ article.update_time }}
          </text>
        </view>

        <!--文章操作-->
        <view
          v-if="me.is_author"
          class="flex  text-df margin-top align-center"
        >
          <button
            class="cu-btn bg-blue shadow"
            @tap="toArticleEditPage(id)"
          >
            修改文章
          </button>
          <button
            class="cu-btn bg-blue shadow margin-left-sm"
            @tap="deleteThisArticle"
          >
            删除文章
          </button>
        </view>

        <!--文章简介-->
        <view class="solid-top solid-bottom margin-top-xl padding-top-sm padding-bottom-sm">
          <text class="text-xl text-grey line">
            {{ article.description }}
          </text>
        </view>

        <!--文章内容-->
        <view
          :data-text="article.content"
          class="margin-top-xl"
        >
          <MarkdownViewer :markdown="article.content" />
        </view>

        <!--文章评论区-->
        <view class="margin-top-xl padding-top-sm solid-top">
          <view class="text-df text-bold">
            评论（{{ comments.length }}条）
          </view>
          <view
            v-for="(item, index) in comments"
            :key="index"
          >
            <view
              v-if="item.parent === 0"
              class="solid-bottom padding-top-sm padding-bottom-sm"
            >
              <!--该评论-->
              <view @tap="reply(item.id)">
                <ArticleComment :comment="item" />
              </view>

              <!--简要展开子评论-->
              <view
                v-for="(kid, index1) in item.kids"
                :key="index1"
                class="text-reply margin-left-xs padding-left-xs"
                @tap="toCommentDetailsPage(item.id)"
              >
                <text
                  class="text-blue"
                  @tap.stop="toUserPage(kid.user.id)"
                >
                  {{ kid.user.nickname }}
                </text>
                <text
                  v-if="kid.parent !== item.id"
                  class="text-blue"
                  @tap.stop="toUserPage(comments[map[kid.parent]].user.id)"
                >
                  @ {{ comments[map[kid.parent]].user.nickname }}
                </text>
                <text>
                  ：{{ kid.content }}
                </text>
              </view>
            </view>
          </view>

          <!--文章评论区底部-->
          <view class="margin-top-sm text-center">
            <text
              v-if="comments.length"
              class="text-grey text-sm"
            >
              没有更多评论啦~
            </text>
            <text
              v-else
              class="text-grey text-sm"
            >
              这里暂时空空如也~
            </text>
          </view>
        </view>
      </view>
    </scroll-view>

    <!--最新评论-->
    <view
      class="cu-bar foot input padding bg-white"
      style="min-height: 120rpx; z-index: 200"
    >
      <!--点赞按钮-->
      <view
        v-if="inEditing === false"
        class="like bg-white"
        @tap="btnLike"
      >
        <text :class="'cuIcon-appreciate bg-white '+(!me.liked ? '' : 'text-blue')">
          {{ article.likes }}
        </text>
      </view>
      <!--评论框-->
      <view class="input-box">
        <input
          v-model="comment"
          :adjust-position="true"
          :placeholder="ph_text"
          style="margin-left: 30rpx"
          @blur="blur"
          @focus="focus"
        >
      </view>
      <button
        class="cu-btn bg-blue shadow"
        style="width: 16vw"
        @tap="createComment"
      >
        发送
      </button>
    </view>
  </view>
</template>

<script>
import {
  createComment, deleteArticle, getArticle, getComments, likeArticle, unlikeArticle,
} from '@/services/article';
import { toArticleCommentPage, toArticleEditPage } from '@/routers/article';
import { toUserPage } from '@/routers/user';
import { defaultMessage } from '@/services/shareMessages';
import ArticleComment from '@/components/ArticleComment.vue';
import MarkdownViewer from '@/components/MarkdownViewer.vue';

export default {
  components: {
    ArticleComment,
    MarkdownViewer,
  },
  data() {
    return {
      article: {
        title: '',
        author: {
          avatar: '',
        },
        publish_time: '',
        update_time: '',
        description: '',
        content: '',
      },
      me: {
        liked: false,
        is_author: false,
      },
      id: 0,
      comments: {},
      map: [],
      parent: 0,
      comment: '',
      inEditing: false,
    };
  },
  onLoad(options) {
    const { id } = options;
    this.id = id;
    this.getArticle(this.id);
  },
  async onShow() {
    await this.getArticle(this.id);
  },
  /**
   * 右上角分享事件
   */
  onShareAppMessage() {
    return {
      title: this.article.title,
      path: `/pages/articles/details?id=${this.id}`,
      ...defaultMessage(),
    };
  },
  computed: {
    ph_text() {
      if (this.parent > 0) {
        const replyUser = this.comments[this.map[this.parent]].user.nickname;
        return `@ ${replyUser}`;
      }
      return '评论...';
    },
  },
  methods: {
    toUserPage,
    toArticleEditPage,

    /**
     * 删除这篇文章
     * @returns {Promise<void>}
     */
    async deleteThisArticle() {
      uni.showModal({
        title: '提示',
        content: '确定删除该文章吗？',
        success: async (res) => {
          if (res.confirm) {
            await deleteArticle(this.id);
            uni.showToast({
              title: '删除成功',
              icon: 'success',
              duration: 2000,
            });
            uni.navigateBack();
          }
        },
      });
    },
    /**
     * 获取文章细节
     * @param id 文章id
     * @returns {Promise<void>}
     */
    async getArticle(id) {
      const res = await getArticle(id);
      if ('me' in res) {
        this.article = res.article;
        this.me = res.me;
        await this.getComments(id);
      }
    },

    /**
     * 获取文章评论
     * @param id 文章id
     */
    async getComments(id) {
      return getComments(id).then((res) => {
        this.comments = res.comments;
        this.map = res.map;
      });
    },

    /**
     * 跳转到评论详情页面
     * @param id 评论id
     */
    toCommentDetailsPage(id) {
      const articleId = this.id;
      const app = getApp();
      app.globalData.comment = this.comments[this.map[id]];
      toArticleCommentPage(articleId, id);
    },

    /**
     * 回复某评论
     * @param id 0 表示直接评论文章，其他表示回复某评论的子评论
     */
    reply(id) {
      this.parent = id;
      this.comment = '';
    },

    /**
     * 发送评论
     */
    createComment() {
      // #ifdef MP-WEIXIN
      uni.showToast({
        title: '当前平台不支持',
        icon: 'none',
        duration: 2000,
      });
      return;
      // #endif
      // eslint-disable-next-line no-unreachable
      const { comment } = this;
      const { parent } = this;
      const id = this.article.id.toString();
      if (comment.length === 0) {
        uni.showToast({
          title: '不能发送空评论',
          icon: 'none',
        });
        return;
      }
      createComment(id, comment, parent).then(async () => {
        await this.getComments(id);
        this.reply(0);
        this.inEditing = false;
        setTimeout(() => {
          uni.showToast({
            title: '发表成功',
          });
        }, 100);
      });
    },

    /**
     * 选中评论框，进入编辑状态
     */
    focus() {
      this.inEditing = true;
    },

    /**
     * 离开评论框，退出编辑状态
     */
    blur() {
      if (this.comment === '') {
        this.reply(0);
      }
      this.inEditing = false;
    },

    /**
     * 点赞按钮
     */
    btnLike() {
      if (!this.me.liked) {
        // 点赞
        likeArticle(this.id).then(() => {
          this.article.likes += 1;
          this.me.liked = true;
        });
      } else {
        // 取消点赞
        uni.showModal({
          title: '取消点赞',
          content: '确定取消点赞吗？',
          success: (res) => {
            if (res.confirm) {
              unlikeArticle(this.id).then(() => {
                this.article.likes -= 1;
                this.me.liked = false;
              });
            }
          },
        });
      }
    },
  },
};
</script>
<style>
page {
  background-color: white;
}

.line {
  line-height: 1.6em;
}

.like {
  background-color: #f5f5f5;
  border-radius: 5000rpx;
  padding: 10rpx 20rpx;
  margin-left: 25rpx;
  font-size: 30rpx;
}

.input-box {
  flex: 1;
  margin-left: 20rpx;
  margin-right: 20rpx;
  border-radius: 5000rpx;
  background-color: #f5f5f5;
}

.border {
  border: 2rpx solid #0081ff;
}

.text-name {
  color: #666666;
  font-size: 30rpx;
}

.text-date {
  color: #9b9b9b;
  font-size: 24rpx;
}

.text-dz {
  color: #999999;
  font-size: 36rpx;
  align-self: center;
}

.text-content {
  position: relative;
  width: 85vw;
  left: 10vw;
  margin-top: 15rpx;
}

.text-reply {
  background-color: #f7f7f7;
  position: relative;
  width: 85vw;
  left: 12vw;
  margin-top: 10rpx;
  padding: 20rpx;
  border-radius: 10rpx;
}
</style>
