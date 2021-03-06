<template>
  <v-data-table
    :headers="headers"
    :items="peers"
    :hide-actions="true"
  >
    <template v-slot:items="row">
      <td class="ip">
        <template v-if="row.item.country_code">
          <img
            v-if="isWindows"
            class="country-flag"
            :title="row.item.country"
            :alt="codeToFlag(row.item.country_code).char"
            :src="codeToFlag(row.item.country_code).url"
          />
          <template v-else>
            {{ codeToFlag(row.item.country_code).char }}
          </template>
        </template>
        {{ row.item.ip }}
        <span class="grey--text">:{{ row.item.port }}</span>
      </td>
      <td>{{ row.item.connection }}</td>
      <td :title="row.item.flags_desc">{{ row.item.flags }}</td>
      <td>{{ row.item.client }}</td>
      <td>{{ row.item.progress | progress }}</td>
      <td>{{ row.item.dl_speed | networkSpeed }}</td>
      <td>{{ row.item.downloaded | networkSize }}</td>
      <td>{{ row.item.up_speed | networkSpeed }}</td>
      <td>{{ row.item.uploaded | networkSize }}</td>
      <td>{{ row.item.relevance | progress }}</td>
      <td>{{ row.item.files }}</td>
    </template>
  </v-data-table>
</template>

<script lang="ts">
import _ from 'lodash';
import Vue from 'vue';
import { codeToFlag, isWindows } from '../../utils';
import Taskable from '@/mixins/taskable';
import { api } from '../../Api';
import { formatSize } from '../../filters';

export default Vue.extend({
  mixins: [Taskable],

  props: {
    hash: String,
    isActive: Boolean,
  },
  data() {
    const headers = [
      { text: 'IP', value: 'ip' },
      { text: '协议', value: 'connection' },
      { text: '标记', value: 'flags' },
      { text: '客户端', value: 'client' },
      { text: '进度', value: 'progress' },
      { text: '下载速度', value: 'dl_speed' },
      { text: '已下载', value: 'downloaded' },
      { text: '上传速度', value: 'up_speed' },
      { text: '已上传', value: 'uploaded' },
      { text: '相关度', value: 'relevance' },
      { text: '文件', value: 'files' },
    ];

    return {
      headers,
      peersObj: null,
      rid: null,
      isWindows,
    };
  },
  filters: {
    networkSpeed(speed: number) {
      if (speed === 0) {
        return null;
      }

      return formatSize(speed) + '/s';
    },
    networkSize(size: number) {
      if (size === 0) {
        return null;
      }

      return formatSize(size);
    },
  },
  computed: {
    peers() {
      return _.map(this.peersObj, (value, key) => {
        return _.merge({}, value, { key });
      });
    },
  },
  methods: {
    codeToFlag(code: string) {
      if (code) {
        return codeToFlag(code);
      }

      return {};
    },
    async getPeers() {
      const resp = await api.getTorrentPeers(this.hash, this.rid);
      this.rid = resp.rid;

      if (resp.full_update) {
        this.peersObj = resp.peers;
      } else {
        const tmp: any = _.cloneDeep(this.peersObj);
        if (resp.peers_removed) {
          for (const key of resp.peers_removed) {
            delete tmp[key];
          }
        }
        this.peersObj = _.merge(tmp, resp.peers);
      }

      if (!this.isActive || this.destroy) {
        return;
      }

      this.task = setTimeout(this.getPeers, 2000);
    },
  },
  async created() {
    if (this.isActive) {
      await this.getPeers();
    }
  },
  watch: {
    async isActive(v) {
      if (v) {
        await this.getPeers();
      } else {
        this.cancelTask();
      }
    }
  },
});
</script>

<style lang="scss" scoped>
::v-deep .ip {
  display: flex;
  align-items: center;

  .country-flag {
    width: 1.5em;
    margin-right: 0.5em;
  }
}
</style>
