<template>

  <v-card class="mx-auto" max-width="1800">
    <v-card-title>Torrents</v-card-title>
    <v-card-text>
      <v-table>
        <thead>
        <tr>
          <th class="text-left">Title</th>
          <th class="text-left">HASH</th>
          <th>Actions</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="item in torrents" :key="item.id">
          <td><a :href="item.url">{{ item.title }}</a> </td>
          <td>{{ item.hash }}</td>
          <td>
            <v-btn color="primary" text @click="deleteTorrent(item)">
              Delete
            </v-btn>
          </td>
        </tr>
        </tbody>
      </v-table>
    </v-card-text>
    <v-card-actions>
      <v-btn
        color="primary"
      >
        Add torrent

        <v-dialog
          v-model="dialogVisible"
          activator="parent"
          width="500"
        >
          <v-card>
            <v-card-title>Add Torrent</v-card-title>
            <v-card-text>
              <v-form @submit.prevent="addTorrent">
                <v-text-field v-model="torrentUrl" label="Torrent URL"></v-text-field>
                <v-select
                  v-model="selectedDownloadPath"
                  :items="downloadPaths"
                  label="Download path"
                  item-text="path"
                  item-value="path"
                  :required="isDownloadPathRequired"
                ></v-select>
                <v-btn type="submit" color="primary" :disabled="loading">
                  <v-progress-circular v-if="loading" indeterminate size="14" class="mr-2"></v-progress-circular>
                  Add
                </v-btn>
              </v-form>
            </v-card-text>
          </v-card>
        </v-dialog>
      </v-btn>
    </v-card-actions>
  </v-card>
  <v-overlay
    v-model="loading"
    absolute="true"
    class="align-center justify-center"
  >
    <v-progress-circular indeterminate size="64"></v-progress-circular>
  </v-overlay>
  <v-dialog v-model="warningDialogVisible" max-width="400">
    <v-card>
      <v-card-title class="headline">Warning</v-card-title>
      <v-card-text>
        Мы попали под ограничение количества скачанных торрентов. Торрент будет добавлен позже автоматически.
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="green darken-1" text @click="warningDialogVisible = false">
          OK
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
  <v-dialog v-model="errorDialogVisible" max-width="400">
    <v-card>
      <v-card-title class="headline">Error</v-card-title>
      <v-card-text>
        Что-то пошло не так. Возможно, линк неверный.
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="green darken-1" text @click="errorDialogVisible = false">
          OK
        </v-btn>
      </v-card-actions>

    </v-card>
  </v-dialog>

</template>

<script>
export default {
  name: "TorrentList",
  data() {
    return {
      torrents: [],
      dialogVisible: false,
      torrentUrl: "",
      warningDialogVisible: false,
      errorDialogVisible: false,
      apiUrl: import.meta.env.VITE_BACKEND_API,
      loading: false,
      downloadPaths: [],
      selectedDownloadPath: null,
    };
  },
  computed: {
    isDownloadPathRequired() {
      return this.downloadPaths.length === 0;
    }
  },
  methods: {
    async getTorrents() {
      const response = await fetch(this.apiUrl + "api/torrents");
      this.torrents = await response.json();
    },
    async getDownloadPaths() {
      const response = await fetch(this.apiUrl + "api/download-paths");
      this.downloadPaths = await response.json();
    },
    async deleteTorrent(item) {
      try {
        const response = await fetch(this.apiUrl + "api/remove", {
          method: "DELETE",
          headers: {
            "Content-Type": "application/json"
          },
          body: JSON.stringify({
            url: item.url,
            hash: item.hash,
          })
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();

        if (data.status === "ok") {
          // Remove the item from the local list
          this.torrents = this.torrents.filter(torrent => torrent.name !== item.name);
        } else {
          console.error("Error removing torrent: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
      }
      // Update the list of torrents
      this.getTorrents();
    },
    async addTorrent() {
      if (this.isDownloadPathRequired && this.selectedDownloadPath === "") {
        this.errorDialogVisible = true;
        throw new Error("Download path is required");
      }
      try {
        this.loading = true;
        const payload = {
          url: this.torrentUrl,
          downloadPath: this.selectedDownloadPath
        };

        const response = await fetch(this.apiUrl + "api/add", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(payload),
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        if (data.status !== 'ok') {
          this.errorDialogVisible = true;
          console.error("Error adding torrent: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
        this.errorDialogVisible = true;
      } finally {
        this.loading = false;
        this.dialogVisible = false;
        this.torrentUrl = "";
        this.getTorrents();
      }
    },

  mounted() {
    this.getTorrents();

    // Create WebSocket url from the current url if it's not set in .env
    this.wsUrl = import.meta.env.VITE_BACKEND_WS || `ws://${window.location.host}/ws`;

    // if https then use wss
    if (window.location.protocol === "https:") {
      this.wsUrl = this.wsUrl.replace("ws:", "wss:");
    }

    this.socket = new WebSocket(this.wsUrl);

    // Connection opened
    this.socket.onmessage = (event) => {
      const msg = event.data;

      // if something was added then update the list of torrents
      if (msg === "added") {
        this.loading = false;
        this.getTorrents();
      } else if (msg === "500") {
        this.loading = false;
        this.errorDialogVisible = true;
      }
    };
  },
  watch: {
    dialogVisible(val) {
      if (!val) {
        this.torrentUrl = "";
      } else {
        this.getDownloadPaths();
      }
    },
  },
};
</script>
