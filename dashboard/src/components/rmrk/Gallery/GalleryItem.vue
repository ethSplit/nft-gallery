 <template>
  <div class="wrapper">
    <div class="tile is-ancestor">
      <div class="tile is-8 is-vertical is-parent">
        <div class="tile is-child box">
            <b-image
              v-if="!isLoading && imageVisible"
              :src="nft.image || require('@/assets/kodadot_logo_v1_transparent_400px.png')"
              :src-fallback="require('@/assets/kodadot_logo_v1_transparent_400px.png')"
              alt="NFT minted image"
              ratio="1by1"
            ></b-image>
            <b-skeleton height="524px" size="is-large" :active="isLoading"></b-skeleton>
          <MediaResolver v-if="nft.animation_url" :class="{ withPicture: imageVisible }" :src="nft.animation_url" :mimeType="mimeType" />
          <Appreciation :accountId="accountId" :currentOwnerId="nft.currentOwner" :nftId="nft.id" />
          <PackSaver v-if="accountId" :accountId="accountId" :currentOwnerId="nft.currentOwner" :nftId="nft.id" />
            <div class="card">
              <div class="card-content">
                <p class="title is-size-2">
                  {{ $t('legend')}}
                </p>
                <p class="subtitle is-size-7">
                  <b v-if="!isLoading"># {{ nftId }}</b>
                  <b-skeleton size="is-large" :active="isLoading"></b-skeleton>
                  <b-tag v-if="nft.price" type="is-dark" size="is-medium">
                    <Money :value="nft.price" :inline="true" />
                  </b-tag>
                  <p v-if="!isLoading" 
                    class="subtitle is-size-5">
                    {{ nft.description }}
                  </p>
                  <b-skeleton :count="3" size="is-large" :active="isLoading"></b-skeleton>
                </p>
              </div>
            </div>
        </div>
      </div>

      <div class="tile is-vertical is-parent">
        <Name :nft="nft" :isLoading="isLoading" />
        <br>
        <b-collapse class="card" animation="slide" 
          aria-id="contentIdForA11y3" :open="false">
          <template #trigger="props">
            <div
              class="card-header"
              role="button"
              aria-controls="contentIdForA11y3">
              <p class="card-header-title">
                {{ $t('actions') }}
              </p>
              <a class="card-header-icon">
                <b-icon
                  :icon="props.open ? 'chevron-up' : 'chevron-down'">
                </b-icon>
              </a>
            </div>
          </template>
          <div class="card-content">
            <div class="content">
              <p class="subtitle">
                <AccountSelect label="Account" v-model="accountId" />
                <AvailableActions :accountId="accountId" :currentOwnerId="nft.currentOwner" :price="nft.price" :nftId="nft.id" />
              </p>
            </div>
          </div>
        </b-collapse>
        <Sharing />
        <br>
        <Facts :nft="nft" />
      </div>
    </div>
  </div>
</template>

<script lang="ts" >
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import { getInstance } from '@/components/rmrk/service/RmrkService';
import { NFTWithMeta, NFT } from '../service/scheme';
import { fetchNFTMetadata, sanitizeIpfsUrl } from '../utils';
import { emptyObject } from '@/utils/empty';
import AccountSelect from '@/components/shared/AccountSelect.vue';
import AvailableActions from './AvailableActions.vue';
import { notificationTypes, showNotification } from '@/utils/notification';
import Money from '@/components/shared/format/Money.vue';
import Sharing from '@/components/rmrk/Gallery/Item/Sharing.vue';
import Facts from '@/components/rmrk/Gallery/Item/Facts.vue';
import Name from '@/components/rmrk/Gallery/Item/Name.vue';
import Appreciation from './Appreciation.vue';
// import MediaResolver from '../Media/MediaResolver.vue'
import api from '@/fetch';
import { resolveMedia } from '../utils';
import { MediaType } from '../types';
import { MetaInfo } from 'vue-meta';

type NFTType =  NFTWithMeta;

@Component<GalleryItem>({
  metaInfo() {
    return {
      meta: [
        { 
          vmid: 'description',
          name: 'description',
          content: '🖼 👀 KodaDot Polkadot/Kusama NFT Market Explorer'
        },
        { property: 'og:type', content: 'website'},
        { property: 'og:title', content: (this.nft.name as string) },
        { property: 'og:description', content: (this.nft.description as string) },
        { property: 'og:image', content: (this.nft.image as string) },
        { property: 'og:video', content: (this.nft.image as string) },
        { property: 'twitter:card', content: 'summary_large_image' },
        { property: 'twitter:site', content: '@KodaDot' },
        { property: 'twitter:title', content: (this.nft.name as string) },
        { property: 'twitter:description', content: (this.nft.description as string) },
        { property: 'twitter:image', content: (this.nft.image as string) },
      ]
    }
  },
  components: {
    AccountSelect,
    AvailableActions,
    Money,
    Sharing,
    Facts,
    Name,
    Appreciation: () => import('./Appreciation.vue'),
    MediaResolver: () => import('../Media/MediaResolver.vue'),
    PackSaver: () => import('../Pack/PackSaver.vue')
  }
})
export default class GalleryItem extends Vue {
  private id: string = '';
  private accountId: string = '';
  private passsword: string = '';
  private nft: NFTType = emptyObject<NFTType>();
  private imageVisible: boolean = true;
  public isLoading: boolean = true;
  public mimeType: string = '';

  public async mounted() {
    this.checkId();
    const rmrkService = getInstance();
    console.log('rmrkService', rmrkService)
    if (!rmrkService || !this.id) {
      return;
    }

    try {
      const nft = await rmrkService.getNFT(this.id);
      console.log(nft);
      const meta = await fetchNFTMetadata(nft);
      console.log(meta);
      this.nft = {
        ...nft,
        ...meta,
        image: sanitizeIpfsUrl(meta.image || ''),
        animation_url: sanitizeIpfsUrl(meta.animation_url || '', 'pinata')
      };
      if (this.nft.animation_url) {
        const { headers } = await api.head(this.nft.animation_url);
        this.mimeType = headers['content-type'];
        const mediaType = resolveMedia(this.mimeType);
        this.imageVisible = ![MediaType.VIDEO, MediaType.IMAGE].some(
          t => t === mediaType
        );
      }
    } catch (e) {
      showNotification(`${e}`, notificationTypes.danger);
      console.warn(e);
    }

    this.isLoading = false;
  }

  public checkId() {
    if (this.$route.params.id) {
      this.id = this.$route.params.id;
    }
  }

  public toast(message: string): void {
    this.$buefy.toast.open(message);
  }

  get nftId() {
    const { id } = this.nft;
    return id;
  }
}
</script>

<style scoped>
.gallery-item__skeleton {
  width: 95%;
  margin: auto;
}

.withPicture {
  margin: 0.75em 0;
}
</style>
