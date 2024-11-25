<template>
  <v-container>
    <v-btn @click="goBack" :disabled="currentPath === rootDirectory">Back</v-btn>
    <div class="list">
      <div v-for="(item, index) in files" :key="index" class="list-item" @click="changeDirectory(item)">
        <div>
          <img v-if="item.type === 'folder'" class="logo" src="./assets/folder.png">
          <img v-if="item.type === 'file'" class="logo" src="./assets/file.png">
        </div>
        <div>{{ item.name }}</div>
        <v-icon v-if="item.type === 'folder'" @click.stop="deleteFolder(item)">mdi-folder-remove</v-icon>
        <v-icon v-if="item.type === 'file'" @click.stop="deleteFile(item)">mdi-file-remove</v-icon>
        <v-icon @click.stop="openRenameDialog(item)">mdi-pencil</v-icon>
      </div>
    </div>
    <div class="create-folder">
      <v-btn @click="openCreateFolderDialog">Create Folder</v-btn>
    </div>
    <div class="upload-file">
      <input type="file" @change="handleFileUpload" />
    </div>

    <v-dialog v-model="createFolderDialog" max-width="500px">
      <v-card>
        <v-card-title>Create Folder</v-card-title>
        <v-card-text>
          <v-text-field v-model="newFolderName" label="Folder Name" required></v-text-field>
        </v-card-text>
        <v-card-actions>
          <v-btn color="primary" text @click="createFolder">Create</v-btn>
          <v-btn color="primary" text @click="createFolderDialog = false">Cancel</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="renameDialog" max-width="500px">
      <v-card>
        <v-card-title>Rename {{ renameItemType }}</v-card-title>
        <v-card-text>
          <v-text-field v-model="newName" label="New Name" required></v-text-field>
        </v-card-text>
        <v-card-actions>
          <v-btn color="primary" text @click="renameItem">Rename</v-btn>
          <v-btn color="primary" text @click="renameDialog = false">Cancel</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-snackbar v-model="snackbar" :timeout="3000" :color="snackbarColor">
      {{ snackbarText }}
      <template v-slot:action="{ attrs }">
        <v-btn text v-bind="attrs" @click="snackbar = false">Close</v-btn>
      </template>
    </v-snackbar>
  </v-container>
</template>

<script>
export default {
  data() {
    return {
      files: [],
      ws: null,
      currentPath: "",
      rootDirectory: "C:/storage",
      newFolderName: "",
      selectedFile: null,
      createFolderDialog: false,
      renameDialog: false,
      renameItemType: "",
      renameItemName: "",
      newName: "",
      snackbar: false,
      snackbarText: "",
      snackbarColor: ""
    };
  },
  methods: {
    connectWebSocket() {
      this.ws = new WebSocket("ws://localhost:9001");

      this.ws.onopen = () => {
        console.log("WebSocket соединение установлено");
        this.fetchFiles();
      };

      this.ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          console.log(data);

          if (data.type === "directory") {
            this.files = data.contents || [];
            this.currentPath = data.path || this.currentPath;
          } else if (data.error) {
            console.error(data.error);
            this.showSnackbar(data.error, "error");
          } else if (data.message) {
            this.showSnackbar(data.message, "success");
            if (data.message === "Directory changed successfully" || data.message === "Renamed successfully" || data.message === "Folder created successfully" || data.message === "File uploaded successfully" || data.message === "Deleted successfully") {
              this.fetchFiles();
            }
          } else if (data.updated_directory) {
            this.files = data.updated_directory.contents || [];
            this.currentPath = data.updated_directory.path || this.currentPath;
          }
        } catch (error) {
          console.error(error);
        }
      };

      this.ws.onerror = (error) => {
        console.error("WebSocket ошибка:", error);
      };

      this.ws.onclose = () => {
        console.log("WebSocket соединение закрыто");
      };
    },

    fetchFiles() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на получение файлов...");
        this.ws.send(JSON.stringify({ action: "list_directory", path: "" }));
      } else {
        console.error("WebSocket не подключен");
      }
    },

    changeDirectory(item) {
      if (item.type === 'folder') {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          console.log("Отправка запроса на смену директории...");
          this.ws.send(JSON.stringify({ action: "change_directory", path: item.name }));
        } else {
          console.error("WebSocket не подключен");
        }
      }
    },

    goBack() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на возврат к родительской директории...");
        this.ws.send(JSON.stringify({ action: "change_directory", path: ".." }));
      } else {
        console.error("WebSocket не подключен");
      }
    },

    deleteFolder(item) {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на удаление папки...");
        this.ws.send(JSON.stringify({ action: "delete", path: item.name }));

        setTimeout(() => {
          this.fetchFiles();
        }, 500);
      } else {
        console.error("WebSocket не подключен");
      }
    },

    deleteFile(item) {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на удаление файла...");
        this.ws.send(JSON.stringify({ action: "delete", path: item.name }));

        setTimeout(() => {
          this.fetchFiles();
        }, 500);
      } else {
        console.error("WebSocket не подключен");
      }
    },

    openCreateFolderDialog() {
      this.createFolderDialog = true;
    },

    createFolder() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на создание папки...");
        this.ws.send(JSON.stringify({ action: "create_folder", name: this.newFolderName }));
        this.newFolderName = "";
        this.createFolderDialog = false;
      } else {
        console.error("WebSocket не подключен");
      }
    },

    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const fileContent = e.target.result;
          if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            console.log("Отправка запроса на загрузку файла...");
            this.ws.send(JSON.stringify({ action: "upload", name: file.name, content: fileContent }));
          } else {
            console.error("WebSocket не подключен");
          }
        };
        reader.readAsBinaryString(file);
      }
    },

    openRenameDialog(item) {
      this.renameItemType = item.type;
      this.renameItemName = item.name;
      this.newName = item.name;
      this.renameDialog = true;
    },

    renameItem() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        console.log("Отправка запроса на переименование...");
        this.ws.send(JSON.stringify({ action: "rename", path: this.renameItemName, new_name: this.newName }));
        this.renameDialog = false;

        setTimeout(() => {
          this.fetchFiles();
        }, 500);
      } else {
        console.error("WebSocket не подключен");
      }
    },

    showSnackbar(message, color) {
      this.snackbarText = message;
      this.snackbarColor = color;
      this.snackbar = true;
    }
  },
  mounted() {
    this.connectWebSocket();
  },
  beforeDestroy() {
    if (this.ws) {
      this.ws.close();
    }
  }
};
</script>

<style>
.list {
  padding-top: 20px;
  padding-bottom: 20px;
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.list-item {
  flex: 0 0 calc(33.33% - 10px);
  box-sizing: border-box;
  margin-bottom: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
}

.list-item:hover {
  background-color: #f0f0f0;
}

.logo {
  width: 42px;
  height: 42px;
}

.create-folder,
.upload-file {
  margin-top: 20px;
}
</style>
