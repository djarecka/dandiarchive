<template>
  <div>
    <v-progress-linear
      v-if="!currentDandiset"
      indeterminate
    />
    <v-container v-else>
      <v-dialog
        v-if="!!itemToDelete"
        v-model="itemToDelete"
        persistent
        max-width="60vh"
      >
        <v-card>
          <v-card-title class="text-h5">
            Really delete this asset?
          </v-card-title>

          <v-card-text>
            Are you sure you want to delete asset <span
              class="font-italic"
            >{{ itemToDelete.name }}</span>?
            <strong>This action cannot be undone.</strong>
          </v-card-text>

          <v-card-actions>
            <v-spacer />
            <v-btn
              @click="itemToDelete = null"
            >
              Cancel
            </v-btn>
            <v-btn
              color="error"
              @click="deleteAsset(itemToDelete)"
            >
              Yes
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

      <v-row>
        <v-col :cols="12">
          <v-card>
            <v-card-title>
              <v-btn
                icon
                exact
                :to="{
                  name: 'dandisetLanding',
                  params: { identifier, version },
                }"
              >
                <v-icon>mdi-home</v-icon>
              </v-btn>
              <v-divider
                vertical
                class="ml-2 mr-3"
              />
              <router-link
                :to="{ name: 'fileBrowser', query: { location: rootDirectory } }"
                style="text-decoration: none;"
                class="mx-2"
              >
                {{ identifier }}
              </router-link>

              <template v-for="(part, i) in splitLocation">
                <template v-if="part">
                  /
                  <router-link
                    :key="part"
                    :to="{ name: 'fileBrowser', query: { location: locationSlice(i) } }"
                    style="text-decoration: none;"
                    class="mx-2"
                  >
                    {{ part }}
                  </router-link>
                </template>
              </template>
              <span class="ml-auto">
                <b>Size</b>
              </span>
            </v-card-title>
            <v-progress-linear
              v-if="$asyncComputed.items.updating"
              indeterminate
            />
            <v-divider v-else />
            <v-list>
              <v-list-item
                v-for="item in items"
                :key="item.name"
                color="primary"
                @click="selectPath(item)"
              >
                <v-icon
                  class="mr-2"
                  color="primary"
                >
                  <template v-if="item.folder">
                    mdi-folder
                  </template>
                  <template v-else>
                    mdi-file
                  </template>
                </v-icon>
                {{ item.name }}
                <v-spacer />

                <v-list-item-action>
                  <v-btn
                    v-if="showDelete(item)"
                    icon
                    @click="itemToDelete = item"
                  >
                    <v-icon color="error">
                      mdi-delete
                    </v-icon>
                  </v-btn>
                </v-list-item-action>

                <v-list-item-action v-if="item.asset_id">
                  <v-btn
                    icon
                    :href="downloadURI(item.asset_id)"
                  >
                    <v-icon color="primary">
                      mdi-download
                    </v-icon>
                  </v-btn>
                </v-list-item-action>

                <v-list-item-action v-if="item.asset_id">
                  <v-btn
                    icon
                    :href="assetMetadataURI(item.asset_id)"
                    target="_blank"
                    rel="noopener"
                  >
                    <v-icon color="primary">
                      mdi-information
                    </v-icon>
                  </v-btn>
                </v-list-item-action>

                <v-list-item-action v-if="item.asset_id">
                  <v-menu
                    bottom
                    left
                  >
                    <template #activator="{ on, attrs }">
                      <v-btn
                        v-if="item.services.length"
                        color="primary"
                        icon
                        v-bind="attrs"
                        v-on="on"
                      >
                        <v-icon>mdi-dots-vertical</v-icon>
                      </v-btn>
                      <v-btn
                        v-else
                        color="primary"
                        disabled
                        icon
                      >
                        <v-icon>mdi-dots-vertical</v-icon>
                      </v-btn>
                    </template>
                    <v-list dense>
                      <v-subheader
                        v-if="item.services.length"
                        class="font-weight-medium"
                      >
                        EXTERNAL SERVICES
                      </v-subheader>

                      <v-list-item
                        v-for="el in item.services"
                        :key="el.name"
                        :href="el.url"
                      >
                        <v-list-item-title class="font-weight-light">
                          {{ el.name }}
                        </v-list-item-title>
                      </v-list-item>
                    </v-list>
                  </v-menu>
                </v-list-item-action>

                <v-list-item-action
                  v-if="item.size"
                  class="justify-end"
                  :style="{width: '4.5em'}"
                >
                  {{ fileSize(item) }}
                </v-list-item-action>
              </v-list-item>
            </v-list>
          </v-card>
        </v-col>
      </v-row>
      <v-pagination
        v-model="page"
        :length="pages"
      />
    </v-container>
  </div>
</template>

<script>
import filesize from 'filesize';

import { dandiRest } from '@/rest';
import store from '@/store';

const parentDirectory = '..';
const rootDirectory = '';

const FILES_PER_PAGE = 15;

const sortByName = (a, b) => {
  if (a.name > b.name) {
    return 1;
  }
  if (b.name > a.name) {
    return -1;
  }
  return 0;
};

const EXTERNAL_SERVICES = [
  {
    name: 'Bioimagesuite/Viewer',
    regex: '.nii(.gz)?$',
    maxsize: 1e9,
    endpoint: 'https://bioimagesuiteweb.github.io/unstableapp/viewer.html?image=',
  },

  {
    name: 'MetaCell/NWBExplorer',
    regex: '.nwb$',
    maxsize: 1e9,
    endpoint: 'http://nwbexplorer.opensourcebrain.org/nwbfile=',
  },
];

export default {
  name: 'FileBrowser',
  props: {
    identifier: {
      type: String,
      required: true,
    },
    version: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      rootDirectory,
      location: rootDirectory,
      owners: [],
      itemToDelete: null,
      page: 1,
      pages: 0,
    };
  },
  computed: {
    splitLocation() {
      return this.location.split('/');
    },

    me() {
      return dandiRest.user ? dandiRest.user.username : null;
    },

    isAdmin() {
      return this.me && dandiRest.user.admin;
    },

    isOwner() {
      return this.me && this.owners.includes(this.me);
    },

    currentDandiset() {
      return store.state.dandiset.dandiset;
    },
  },
  asyncComputed: {
    items: {
      async get() {
        const {
          version, identifier, location, page,
        } = this;

        const { folders, files, count } = await dandiRest.assetPaths(
          identifier, version, location, page, FILES_PER_PAGE,
        );
        this.owners = (await dandiRest.owners(identifier)).data
          .map((x) => x.username);
        this.pages = Math.ceil(count / FILES_PER_PAGE);

        return [
          ...location !== rootDirectory ? [{ name: parentDirectory, folder: true }] : [],
          ...Object.keys(folders).map(
            (key) => ({ ...folders[key], name: `${key}/`, folder: true }),
          ).sort(sortByName),
          ...Object.keys(files).map(
            (key) => {
              const { asset_id, size } = files[key];
              const services = this.getExternalServices(asset_id, key, size);
              return {
                ...files[key],
                name: key,
                folder: false,
                services,
              };
            },
          ).sort(sortByName),
        ];
      },
      default: null,
    },
  },
  watch: {
    location(location) {
      const { location: existingLocation } = this.$route.query;

      // Reset page to 1 when location changes
      this.page = 1;

      // Update route when location changes
      if (existingLocation === location) { return; }
      this.$router.push({
        ...this.$route,
        query: { location },
      });
    },
    items(items) {
      if (items && !items.length) {
        // If the API call returns no items, go back to the root (shouldn't normally happen)
        this.location = rootDirectory;
      }
    },
    $route: {
      immediate: true,
      handler(route) {
        this.location = route.query.location || rootDirectory;
      },
    },
  },
  async created() {
    // Don't extract currentDandiset, for reactivity
    const { identifier, version } = this;
    if (!this.currentDandiset) {
      this.fetchDandiset({ identifier, version });
    }
  },
  methods: {
    locationSlice(index) {
      return `${this.splitLocation.slice(0, index + 1).join('/')}/`;
    },
    selectPath(item) {
      const { name, folder } = item;

      if (!folder) { return; }
      if (name === parentDirectory) {
        const slicedLocation = this.location.split('/').slice(0, -2);
        this.location = slicedLocation.length ? `${slicedLocation.join('/')}/` : '';
      } else {
        this.location = `${this.location}${name}`;
      }
    },

    downloadURI(asset_id) {
      return dandiRest.assetDownloadURI(this.identifier, this.version, asset_id);
    },

    getExternalServices(asset_id, name, size) {
      const { identifier, version } = this;
      return EXTERNAL_SERVICES
        .filter((service) => new RegExp(service.regex).test(name) && size <= service.maxsize)
        .map((service) => ({
          name: service.name,
          url: `${service.endpoint}${dandiRest.assetDownloadURI(identifier, version, asset_id)}`,
        }));
    },

    assetMetadataURI(asset_id) {
      return dandiRest.assetMetadataURI(this.identifier, this.version, asset_id);
    },

    fileSize(item) {
      return filesize(item.size, { round: 1, base: 10, standard: 'iec' });
    },

    showDelete(item) {
      return this.version === 'draft' && !item.folder && (this.isAdmin || this.isOwner);
    },

    async deleteAsset(item) {
      const { asset_id } = item;
      if (asset_id !== undefined) {
        // Delete the asset on the server.
        await dandiRest.deleteAsset(this.identifier, this.version, asset_id);

        // Recompute the items to display in the browser.
        this.$asyncComputed.items.update();
      }
      this.itemToDelete = null;
    },

    fetchDandiset() {
      store.dispatch.dandiset.fetchDandiset({
        identifier: this.identifier,
        version: this.version,
      });
    },
  },
};
</script>
