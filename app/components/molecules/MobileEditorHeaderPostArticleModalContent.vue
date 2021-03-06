<template>
  <div class="modal-body">
    <div class="wrapper">
      <h3 class="headline">
        1. サムネイルの選択
      </h3>
      <div class="thumbnails">
        <span v-if="suggestedThumbnails.length === 0" class="no-thumbnail-message">
          画像がありません
        </span>
        <div
          v-for="img in suggestedThumbnails"
          :key="img"
          class="thumbnail-box"
          :class="{ selected: img === thumbnail }"
          @click.prevent="selectThumbnail"
        >
          <img :src="img" class="thumbnail">
        </div>
      </div>
      <h3 class="headline">
        2. カテゴリの設定
      </h3>
      <div class="article-type-select-box">
        <no-ssr>
          <select
            required
            class="article-type-select"
            :value="topicType"
            @change="handleChangeTopicType"
          >
            <option value="" disabled selected class="placeholder">
              選択してください
            </option>
            <option v-for="topic in topics" :value="topic.name">
              {{ topic.display_name }}
            </option>
          </select>
        </no-ssr>
      </div>
      <h3 class="headline">
        3. タグの設定
      </h3>
      <tags-input-form @change-tag-validation-state="onChangeTagValidationState" />
      <app-button
        class="submit"
        :disabled="!publishable || isInvalidTag || publishingArticle"
        @click="publish"
      >
        公開する
      </app-button>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import { ADD_TOAST_MESSAGE } from 'vuex-toast'
import AppButton from '../atoms/AppButton'
import TagsInputForm from '../molecules/TagsInputForm'

export default {
  components: {
    AppButton,
    TagsInputForm
  },
  data() {
    return {
      publishingArticle: false,
      isThumbnailSelected: false,
      isInvalidTag: false
    }
  },
  async created() {
    try {
      await this.getTopics()
    } catch (error) {
      this.sendNotification({
        text: 'エラーが発生しました。しばらく時間を置いて再度お試しください',
        type: 'warning'
      })
    }
  },
  methods: {
    async publish() {
      try {
        // 「公開する」ボタンを押した瞬間はisInvalidTagの値が更新されないため、
        // isInvalidTagの状態が更新されるまで待つ
        await this.$nextTick()

        if (!this.publishable || this.isInvalidTag) return
        this.publishingArticle = true
        const { articleId, title, body, topicType } = this
        const hasTitle = title !== undefined && title !== null && title !== ''
        const hasBody = body !== '<p>&nbsp;</p>'
        if (!hasTitle) this.sendNotification({ text: 'タイトルを入力してください' })
        if (!hasBody) this.sendNotification({ text: '本文にテキストを入力してください' })
        if (topicType === null) this.sendNotification({ text: 'カテゴリを選択してください' })
        if (!hasTitle || !hasBody || topicType === null) {
          this.publishingArticle = false
          return
        }

        const articleTitle = { title }
        const articleBody = { body }
        // タグのデータ形式をAPIに適するように整形
        const tags = this.tags.map(tag => tag.text)

        if (
          location.href.includes('/me/articles/draft') ||
          location.href.includes('/me/articles/new')
        ) {
          await this.putDraftArticleTitle({ articleTitle, articleId })
          await this.putDraftArticleBody({ articleBody, articleId })
          await this.publishDraftArticleWithHeader({
            articleId,
            topic: topicType,
            tags,
            eyeCatchUrl: this.thumbnail
          })
        } else if (location.href.includes('/me/articles/public')) {
          await this.putPublicArticleTitle({ articleTitle, articleId })
          await this.putPublicArticleBody({ articleBody, articleId })
          await this.republishPublicArticleWithHeader({
            articleId,
            topic: topicType,
            tags,
            eyeCatchUrl: this.thumbnail
          })
        }
        this.setMobileEditorHeaderPostArticleModal({ isShow: false })
        this.$router.push(`/${this.currentUserInfo.user_id}/articles/${articleId}`)
        this.sendNotification({ text: '記事を公開しました' })
        this.resetArticleTopic()

        if (!this.currentUserInfo.is_created_article) {
          this.setFirstProcessModal({ isShow: true })
          this.setFirstProcessCreatedArticleModal({ isShow: true })
        }
      } catch (e) {
        this.publishingArticle = false
        this.sendNotification({ text: '記事の公開に失敗しました', type: 'warning' })
        console.error(e)
      }
    },
    selectThumbnail({ target }) {
      this.isThumbnailSelected = true
      this.updateThumbnail({ thumbnail: target.src === this.thumbnail ? '' : target.src })
    },
    handleChangeTopicType(event) {
      this.topic = event.target.value
      this.setArticleTopic({ topicType: this.topic })
    },
    onChangeTagValidationState(isInvalid) {
      this.isInvalidTag = isInvalid
    },
    ...mapActions({
      sendNotification: ADD_TOAST_MESSAGE
    }),
    ...mapActions('article', [
      'updateThumbnail',
      'publishDraftArticleWithHeader',
      'republishPublicArticleWithHeader',
      'putDraftArticle',
      'putDraftArticleTitle',
      'putDraftArticleBody',
      'putPublicArticleTitle',
      'putPublicArticleBody',
      'updateSuggestedThumbnails',
      'postArticleImage',
      'updateBody',
      'setIsSaving',
      'getTopics',
      'resetArticleTopic',
      'setArticleTopic'
    ]),
    ...mapActions('user', [
      'setFirstProcessModal',
      'setFirstProcessCreatedArticleModal',
      'setMobileEditorHeaderPostArticleModal'
    ])
  },
  computed: {
    publishable() {
      return (!this.isEditedTitle || !this.isEditedBody) && !this.isSaving
    },
    ...mapGetters('article', [
      'articleId',
      'title',
      'body',
      'thumbnail',
      'suggestedThumbnails',
      'isSaving',
      'isEditedTitle',
      'isEditedBody',
      'topics',
      'topicType',
      'tags'
    ]),
    ...mapGetters('user', ['currentUserInfo'])
  },
  watch: {
    suggestedThumbnails() {
      if (this.thumbnail !== '') return
      if (!this.isThumbnailSelected) {
        this.updateThumbnail({ thumbnail: this.suggestedThumbnails[0] })
        return
      }
      if (
        this.isThumbnailSelected &&
        Array.from(document.querySelectorAll('.thumbnails img')).filter(
          img => img.classList.contains('selected').length !== 0
        )
      ) {
        this.updateThumbnail({ thumbnail: this.suggestedThumbnails[0] })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.modal-body {
  margin: 0 auto;
  display: flex;
  justify-content: center;
}

.wrapper {
  align-self: center;
  background-color: #ffffff;
  border-radius: 4px;
  box-sizing: border-box;
  display: flex;
  flex-flow: column nowrap;
  justify-content: center;
  padding: 40px;
  position: relative;
  width: 340px;

  .headline {
    color: #000000;
    font-size: 14px;
    font-weight: 500;
    letter-spacing: 0.8px;
    margin: 0 0 10px;
    text-align: left;
  }

  .thumbnails {
    overflow-x: scroll;
    overflow-y: hidden;
    white-space: nowrap;
    margin-bottom: 10px;
    user-select: none;
    height: 120px;

    &::-webkit-scrollbar {
      height: 40px;
    }

    &::-webkit-scrollbar-thumb {
      /* スクロールバーをドラッグしやすくするため、スクロールバー領域を広めにとる */
      background: linear-gradient(
        0deg,
        transparent 0%,
        transparent 75%,
        #0086cc 75%,
        #0086cc 80%,
        transparent 80%,
        transparent 100%
      );
    }

    .no-thumbnail-message {
      font-size: 14px;
      margin-top: 40px;
      display: block;
    }

    .thumbnail-box {
      display: inline-block;
      box-sizing: border-box;
      width: 80px;
      margin-right: 20px;
      height: 80px;

      .thumbnail {
        box-sizing: border-box;
        cursor: pointer;
        height: 80px;
        width: 80px;
        object-fit: cover;
      }

      &:last-child {
        margin-right: 0;
      }
    }

    .selected {
      position: relative;

      &:before {
        background-color: rgba(2, 134, 204, 0.5);
        content: ' ';
        cursor: pointer;
        display: block;
        height: 100%;
        left: 0;
        position: absolute;
        top: 0;
        width: 80px;
        z-index: 0;
      }

      &:after {
        bottom: 8px;
        color: #fff;
        content: '選択中';
        font-size: 14px;
        font-weight: bold;
        left: 0;
        letter-spacing: 0.8px;
        position: absolute;
        right: 0;
        text-align: center;
      }
    }
  }

  .article-type-select-box {
    box-shadow: 0 0 8px 0 rgba(192, 192, 192, 0.5);
    margin-bottom: 40px;
    padding: 6px 8px;
    position: relative;

    &::after,
    &::before {
      position: absolute;
      right: 8px;
      width: 0;
      height: 0;
      padding: 0;
      content: '';
      border-left: 6px solid transparent;
      border-right: 6px solid transparent;
      pointer-events: none;
    }

    &::after {
      top: 7px;
      border-bottom: 8px solid #0086cc;
    }

    &::before {
      top: 17px;
      border-top: 8px solid #0086cc;
    }

    .article-type-select {
      appearance: none;
      background-image: none;
      background: transparent;
      border: none;
      box-shadow: none;
      color: #000;
      cursor: pointer;
      font-size: 14px;
      outline: none;
      padding-right: 1em;
      text-indent: 0.01px;
      text-overflow: ellipsis;
      width: 100%;

      .placeholder {
        display: none;
      }

      &::-ms-expand {
        display: none;
      }
    }
  }

  .submit {
    margin: 0 auto;
  }
}

@-moz-document url-prefix() {
  .article-type-select-box {
    &::after {
      top: 15px !important;
    }

    &::before {
      top: 25px !important;
    }
  }
}
</style>
