<template>
  <div class="home">
    <div class="header">
      <div class="buttons">
        <span class="icon is-midium">
          <button @click="addFolder">
            <i class="far fa-2x fa-folder"></i>
          </button>
        </span>
        <span class="icon is-midium show-mobile">
          <button @click="toggleFolders">
            <i class="fas fa-2x fa-sliders-h"></i>
          </button>
        </span>
        <span class="icon is-midium">
          <button @click="addMemo">
            <i class="fas fa-2x fa-edit"></i>
          </button>
        </span>
        <span class="icon is-midium">
          <button @click="deleteFolderOrMemo">
            <i class="far fa-2x fa-trash-alt"></i>
          </button>
        </span>
      </div>
      <div>
        <button v-if="user" @click="logout">ログアウト</button>
        <router-link v-else to="/login" tag="button">ログイン</router-link>
      </div>
    </div>

    <div class="mainContent">
      <div class="foldersWrapper hidden-mobile" :class="{ fullscreen: showFolders }">
        <div v-if="folders.length">
          <div
            class="folder-list"
            v-for="(folder, index) in folders"
            @click="selectFolder(index)"
            :focused="focusing == 'folder'"
            :data-selected="index === selectedFolderIndex"
            v-bind:key="index"
            @keydown.prevent.up.exact="movePreviousFolder"
            @keydown.prevent.down.exact="moveNextFolder"
            @keydown.prevent.right.exact="focusMemos"
            tabindex="0"
          >
            <p>{{ folder.title }}</p>
          </div>
        </div>
      </div>
      <div class="memosWrapper">
        <div v-if="selectedMemos.length">
          <div
            class="memo-list"
            v-for="(memo, index) in selectedMemos"
            @click="selectMemo(index)"
            :focused="focusing == 'memo'"
            :data-selected="index == selectedMemoIndex"
            v-bind:key="index"
            @keydown.prevent.up.exact="movePreviousMemo"
            @keydown.prevent.down.exact="moveNextMemo"
            @keydown.prevent.left.exact="focusFolders"
            @keydown.prevent.right.exact="focusEditor"
            tabindex="0"
          >
            <p class="memo-title">{{ memo.markdown | title }}</p>
            <p class="memo-digest">{{ memo.updateDate }} {{ memo.markdown | digest }}</p>
          </div>
        </div>
      </div>
      <div v-if="selectedMemos.length" class="editorWrapper">
        <p class="update-date">{{ selectedMemos[selectedMemoIndex].updateDate }}</p>
        <textarea
          id="editor"
          class="editor"
          placeholder="todo"
          @click="selectEditor"
          v-model="selectedMemos[selectedMemoIndex].markdown"
        />
      </div>
    </div>
  </div>
</template>

<script>
import auth from "../utils/auth";
import moment from "moment";

/* global firebase */

export default {
  data() {
    return {
      user: null,
      folders: [
        {
          title: "New Folder",
          memos: [
            {
              updateDate: "null",
              markdown: ""
            }
          ]
        }
      ],
      showFolders: false,
      selectedFolderIndex: 0,
      selectedMemoIndex: 0,
      focusing: null // folder, memo or editor
    };
  },
  created() {
    auth.getUser().then(user => {
      if (user) {
        this.user = user;
        firebase
          .database()
          .ref("memos/" + this.user.uid)
          .once("value")
          .then(result => {
            if (result.val()) {
              this.folders = result.val();
            }
          });
      }
    });
  },
  watch: {
    selectedMemos: {
      handler(val) {
        if (val[this.selectedMemoIndex]) {
          val[this.selectedMemoIndex].updateDate = moment().format(
            "YYYY-MM-DD HH:mm"
          );
          this.saveMemos();
        }
      },
      deep: true
    }
  },
  filters: {
    title(markdown) {
      return markdown !== undefined && markdown != ""
        ? markdown.trim().split("\n")[0]
        : "New Note";
    },
    digest(markdown) {
      const digest = markdown.trim().split("\n")[1];
      return digest !== undefined && digest !== ""
        ? digest
        : "No additional text";
    }
  },
  computed: {
    selectedMemos() {
      return this.folders[this.selectedFolderIndex].memos;
    }
  },
  methods: {
    logout() {
      auth.logout();
    },
    addMemo() {
      this.selectedMemos.push({
        updateDate: moment().format("YYYY-MM-DD HH:mm"),
        markdown: ""
      });
      this.selectedMemoIndex = this.selectedMemos.length - 1;
    },
    addFolder() {
      this.folders.push({
        title: "New Folder",
        memos: [
          {
            updateDate: null,
            markdown: ""
          }
        ]
      });
      this.selectedFolderIndex = this.folders.length - 1;
    },
    deleteFolderOrMemo() {
      if (this.focusing == "folder") {
        const result = confirm("フォルダーを消しても良いですか？");
        if (result) {
          this.folders.splice(this.selectedFolderIndex, 1);
          if (this.selectedFolderIndex > 0) {
            this.selectedFolderIndex--;
          }
        }
      } else if (this.focusing == "memo" || this.focusing == "editor") {
        this.selectedMemos.splice(this.selectedMemoIndex, 1);
        if (this.selectedMemoIndex > 0) {
          this.selectedMemoIndex--;
        }
      }
      this.saveMemos();
    },
    selectFolder(index) {
      this.selectedFolderIndex = index;
      this.selectedMemoIndex = 0;
      this.focusing = "folder";
      this.showFolders = false;
    },
    selectMemo(index) {
      this.selectedMemoIndex = index;
      this.focusing = "memo";
    },
    selectEditor() {
      this.focusing = "editor";
    },
    saveMemos() {
      if (this.user)
        firebase
          .database()
          .ref("memos/" + this.user.uid)
          .set(this.folders);
    },
    toggleFolders() {
      this.showFolders = !this.showFolders;
    },
    movePreviousFolder(event) {
      if (this.selectedFolderIndex !== 0) {
        this.selectFolder(this.selectedFolderIndex - 1);
        event.target.previousSibling.focus();
      }
    },
    moveNextFolder(event) {
      if (this.selectedFolderIndex !== this.folders.length - 1) {
        this.selectFolder(this.selectedFolderIndex + 1);
        event.target.nextSibling.focus();
      }
    },
    movePreviousMemo(event) {
      if (this.selectedMemoIndex !== 0) {
        this.selectMemo(this.selectedMemoIndex - 1);
        event.target.previousSibling.focus();
      }
    },
    moveNextMemo(event) {
      if (
        this.selectedMemoIndex !==
        this.folders[this.selectedFolderIndex].memos.length - 1
      ) {
        this.selectMemo(this.selectedMemoIndex + 1);
        event.target.nextSibling.focus();
      }
    },
    focusFolders() {
      this.focusing = "folder";
      document.querySelectorAll('[data-selected="true"]')[0].focus();
    },
    focusMemos() {
      this.focusing = "memo";
      document.querySelectorAll('[data-selected="true"]')[1].focus();
    },
    focusEditor() {
      document.getElementById("editor").focus();
    }
  }
};
</script>

<style lang="scss" scoped>
.home {
  height: 100%;
}
.header {
  display: flex;
  height: 10%;
  padding: 10px;
  justify-content: space-around;
  border: solid 1px gray;
  background-color: #dad9da;
}
.mainContent {
  height: 90%;
  position: relative;
  display: flex;
  align-items: start;
}
.foldersWrapper {
  width: 20rem;
  height: 100%;
  background-color: #dad9da;
  border: solid 1px silver;
  border-top: none;
}
.folder-list {
  &[data-selected="true"] {
    background-color: #979797;
    &[focused="true"] {
      color: white;
      background-color: #0b5ad5;
    }
  }
}
.memosWrapper {
  width: 20rem;
}
.memo-list {
  &[data-selected="true"] {
    background-color: #e1e0e0;
    &[focused="true"] {
      background-color: #fde36d;
    }
  }
}
.memo-digest {
  font-size: 0.8rem;
  color: gray;
}
.buttons {
  flex-grow: 1;
  .icon {
    margin: 0 20px;
  }
}
.editorWrapper {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  border: solid 1px silver;
  border-top: none;
  padding: 4px;
  > div {
    height: 100%;
  }
  .update-date {
    font-size: 0.8rem;
    color: gray;
  }
  .editor {
    font-size: 1rem;
    flex-grow: 1;
    resize: none;
    border: none;
  }
  .editor:focus {
    outline: none !important;
  }
}
.icon {
  button {
    box-sizing: content-box;
    border-radius: 10%;
    border-color: white;
  }
}
.show-mobile {
  display: none;
}
@media screen and (max-width: 480px) {
  .hidden-mobile {
    display: none;
  }
  .show-mobile {
    display: inherit;
  }
  .fullscreen {
    display: block;
    position: absolute;
    width: 100%;
  }
}
</style>
