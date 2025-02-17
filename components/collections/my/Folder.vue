<template>
  <div>
    <div
      :class="['row-wrapper transition duration-150 ease-in-out', { 'bg-bgDarkColor': dragging }]"
      @dragover.prevent
      @drop.prevent="dropEvent"
      @dragover="dragging = true"
      @drop="dragging = false"
      @dragleave="dragging = false"
      @dragend="dragging = false"
    >
      <div>
        <button class="icon" @click="toggleShowChildren">
          <i class="material-icons" v-show="!showChildren && !isFiltered">arrow_right</i>
          <i class="material-icons" v-show="showChildren || isFiltered">arrow_drop_down</i>
          <i v-if="isSelected" class="text-green-400 material-icons">check_circle</i>
          <i v-else class="material-icons">folder_open</i>
          <span>{{ folder.name ? folder.name : folder.title }}</span>
        </button>
      </div>
      <v-popover v-if="!saveRequest">
        <button class="tooltip-target icon" v-tooltip.left="$t('more')">
          <i class="material-icons">more_vert</i>
        </button>
        <template slot="popover">
          <div>
            <button
              class="icon"
              @click="$emit('add-folder', { folder, path: folderPath })"
              v-close-popover
            >
              <i class="material-icons">create_new_folder</i>
              <span>{{ $t("new_folder") }}</span>
            </button>
          </div>
          <div>
            <button
              class="icon"
              @click="$emit('edit-folder', { folder, folderIndex, collectionIndex })"
              v-close-popover
            >
              <i class="material-icons">edit</i>
              <span>{{ $t("edit") }}</span>
            </button>
          </div>
          <div>
            <button class="icon" @click="confirmRemove = true" v-close-popover>
              <i class="material-icons">delete</i>
              <span>{{ $t("delete") }}</span>
            </button>
          </div>
        </template>
      </v-popover>
    </div>
    <div v-show="showChildren || isFiltered">
      <ul class="flex-col">
        <li
          v-for="(subFolder, subFolderIndex) in folder.folders"
          :key="subFolder.name"
          class="ml-8 border-l border-brdColor"
        >
          <CollectionsMyFolder
            :folder="subFolder"
            :folder-index="subFolderIndex"
            :collection-index="collectionIndex"
            :doc="doc"
            :saveRequest="saveRequest"
            :collectionsType="collectionsType"
            :folder-path="`${folderPath}/${subFolderIndex}`"
            :picked="picked"
            @add-folder="$emit('add-folder', $event)"
            @edit-folder="$emit('edit-folder', $event)"
            @edit-request="$emit('edit-request', $event)"
            @update-team-collections="$emit('update-team-collections')"
            @select="$emit('select', $event)"
            @remove-request="removeRequest"
          />
        </li>
      </ul>
      <ul class="flex-col">
        <li
          v-for="(request, index) in folder.requests"
          :key="index"
          class="flex ml-8 border-l border-brdColor"
        >
          <CollectionsMyRequest
            :request="request"
            :collection-index="collectionIndex"
            :folder-index="folderIndex"
            :folder-name="folder.name"
            :request-index="index"
            :folder-path="folderPath"
            :doc="doc"
            :picked="picked"
            :saveRequest="saveRequest"
            :collectionsType="collectionsType"
            @edit-request="$emit('edit-request', $event)"
            @select="$emit('select', $event)"
            @remove-request="removeRequest"
          />
        </li>
      </ul>
      <ul
        v-if="
          folder.folders &&
          folder.folders.length === 0 &&
          folder.requests &&
          folder.requests.length === 0
        "
      >
        <li class="flex ml-8 border-l border-brdColor">
          <p class="info"><i class="material-icons">not_interested</i> {{ $t("folder_empty") }}</p>
        </li>
      </ul>
    </div>
    <SmartConfirmModal
      :show="confirmRemove"
      :title="$t('are_you_sure_remove_folder')"
      @hide-modal="confirmRemove = false"
      @resolve="removeFolder"
    />
  </div>
</template>

<script>
import { fb } from "~/helpers/fb"
import { getSettingSubject } from "~/newstore/settings"

export default {
  name: "folder",
  props: {
    folder: Object,
    folderIndex: Number,
    collectionIndex: Number,
    folderPath: String,
    doc: Boolean,
    saveRequest: Boolean,
    isFiltered: Boolean,
    collectionsType: Object,
    picked: Object,
  },
  data() {
    return {
      showChildren: false,
      dragging: false,
      confirmRemove: false,
      prevCursor: "",
      cursor: "",
    }
  },
  subscriptions() {
    return {
      SYNC_COLLECTIONS: getSettingSubject("syncCollections"),
    }
  },
  computed: {
    isSelected() {
      return (
        this.picked &&
        this.picked.pickedType === "my-folder" &&
        this.picked.folderPath === this.folderPath
      )
    },
  },
  methods: {
    syncCollections() {
      if (fb.currentUser !== null && this.SYNC_COLLECTIONS) {
        fb.writeCollections(
          JSON.parse(JSON.stringify(this.$store.state.postwoman.collections)),
          "collections"
        )
      }
    },
    toggleShowChildren() {
      if (this.$props.saveRequest)
        this.$emit("select", {
          picked: {
            pickedType: "my-folder",
            collectionIndex: this.collectionIndex,
            folderName: this.folder.name,
            folderPath: this.folderPath,
          },
        })
      this.showChildren = !this.showChildren
    },
    removeFolder() {
      this.$store.commit("postwoman/removeFolder", {
        collectionIndex: this.$props.collectionIndex,
        folderName: this.$props.folder.name,
        folderIndex: this.$props.folderIndex,
        flag: "rest",
      })
      this.syncCollections()
      this.$toast.error(this.$t("deleted"), {
        icon: "delete",
      })
    },
    dropEvent({ dataTransfer }) {
      this.dragging = !this.dragging
      const oldCollectionIndex = dataTransfer.getData("oldCollectionIndex")
      const oldFolderIndex = dataTransfer.getData("oldFolderIndex")
      const oldFolderName = dataTransfer.getData("oldFolderName")
      const requestIndex = dataTransfer.getData("requestIndex")
      const flag = "rest"

      this.$store.commit("postwoman/moveRequest", {
        oldCollectionIndex,
        newCollectionIndex: this.$props.collectionIndex,
        newFolderIndex: this.$props.folderIndex,
        newFolderName: this.$props.folder.name,
        oldFolderIndex,
        oldFolderName,
        requestIndex,
        flag,
      })
      this.syncCollections()
    },
    removeRequest({ collectionIndex, folderName, requestIndex }) {
      this.$emit("remove-request", {
        collectionIndex,
        folderName,
        requestIndex,
      })
    },
  },
}
</script>
