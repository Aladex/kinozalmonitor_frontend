<template>
  <v-card class="mx-auto" max-width="1800">
    <v-card-title>Torrents</v-card-title>
    <v-card-text>
      <v-table>
        <thead>
        <tr>
          <th class="text-left">Title</th>
          <th class="text-left">Name</th>
          <th class="text-left">HASH</th>
          <th class="text-left">Url</th>
          <th>Actions</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="item in torrents" :key="item.id">
          <td>{{ item.title }}</td>
          <td>{{ item.name }}</td>
          <td>{{ item.hash }}</td>
          <td>{{ item.url }}</td>
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
      apiUrl: import.meta.env.VITE_BACKEND_API,
      loading: false,
    };
  },
  methods: {
    async getTorrents() {
      const response = await fetch(this.apiUrl + "api/torrents");
      this.torrents = await response.json();
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
      try {
        this.loading = true;
        const response = await fetch(this.apiUrl + "api/add", {
          method: "POST",
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
          },
          body: `url=${encodeURIComponent(this.torrentUrl)}`,
        });

        if (!response.ok) {
          this.warningDialogVisible = true;
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();

        if (data.status === "ok") {
          // Обновить список торрентов после успешного добавления
          this.getTorrents();
        } else {
          console.error("Error adding torrent: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
      } finally {
        this.loading = false;
        this.getTorrents();
      }

      // Сбросить значение поля URL и закрыть диалоговое окно
      this.torrentUrl = "";
      this.dialogVisible = false;
    },
  },
  mounted() {
    this.getTorrents();
  },
};
</script>
