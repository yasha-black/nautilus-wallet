<template>
  <div class="flex flex-col gap-4">
    <div>
      <input
        type="text"
        :disabled="loading"
        v-model="filter"
        placeholder="Search"
        class="w-full control block"
      />
    </div>
    <div class="border rounded">
      <table class="table">
        <thead>
          <tr>
            <th colspan="2">Asset</th>
            <th>
              <div class="flex-row justify-end flex gap-2 align-middle">
                <tool-tip
                  :label="hideBalances ? 'Show' : 'Hide'"
                  tip-class="normal-case"
                  class="align-middle"
                >
                  <a class="cursor-pointer inline-flex" @click="toggleHideBalance()">
                    <mdi-icon :name="hideBalances ? 'eye-off' : 'eye'" size="16" />
                  </a>
                </tool-tip>
                Balance
              </div>
            </th>
          </tr>
        </thead>
        <tbody>
          <template v-if="loading">
            <tr v-for="i in prevCount" :key="i">
              <td class="w-14 align-middle">
                <empty-logo class="h-8 w-8 animate-pulse fill-gray-300" />
              </td>
              <td class="align-middle">
                <div class="skeleton h-3 w-2/3 rounded"></div>
              </td>
              <td class="text-right w-50 align-middle">
                <div class="skeleton h-3 w-3/5 rounded"></div>
                <template v-if="i === 1">
                  <br />
                  <div class="skeleton h-3 w-2/5 rounded"></div>
                </template>
              </td>
            </tr>
          </template>
          <tr v-else v-for="asset in assets" :key="asset.tokenId" class="h-19">
            <td class="w-14 min-w-14 align-middle">
              <asset-icon
                class="h-8 w-8 align-middle"
                :token-id="asset.tokenId"
                :type="asset.info?.type"
              />
            </td>
            <td class="align-middle">
              <p v-if="isErg(asset.tokenId)" class="font-semibold">
                {{ asset.info?.name }}
              </p>
              <a v-else @click="selectedAsset = asset" class="break-anywhere cursor-pointer">
                <template v-if="asset.info?.name">{{
                  $filters.compactString(asset.info?.name, 40)
                }}</template>
                <template v-else>{{ $filters.compactString(asset.tokenId, 12) }}</template>
              </a>
            </td>
            <td class="text-right align-middle whitespace-nowrap">
              <div v-if="hideBalances" class="flex flex-col gap-1 items-end">
                <div class="skeleton animate-none h-4.5 w-full rounded"></div>
                <div class="skeleton animate-none h-3 w-3/4 rounded"></div>
              </div>
              <template v-else>
                <p>
                  {{ $filters.formatBigNumber(asset.confirmedAmount) }}
                </p>
                <tool-tip
                  v-if="!asset.confirmedAmount.isZero() && ergPrice && rate(asset.tokenId)"
                  :label="`1 ${asset.info?.name} <br /> ≈ ${$filters.formatBigNumber(
                    price(asset.tokenId),
                    2
                  )} ${$filters.uppercase(conversionCurrency)}`"
                >
                  <p class="text-xs text-gray-500">
                    ≈
                    {{
                      $filters.formatBigNumber(
                        asset.confirmedAmount.multipliedBy(price(asset.tokenId)),
                        2
                      )
                    }}
                    {{ $filters.uppercase(conversionCurrency) }}
                  </p>
                </tool-tip>
              </template>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <asset-info-modal
      @close="selectedAsset = undefined"
      :token-id="selectedAsset?.tokenId"
      :confirmed-balance="selectedAsset?.confirmedAmount"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { GETTERS } from "@/constants/store/getters";
import { ERG_TOKEN_ID } from "@/constants/ergo";
import { StateAsset } from "@/types/internal";
import BigNumber from "bignumber.js";
import EmptyLogo from "@/assets/images/tokens/asset-nft-picture.svg";
import AssetInfoModal from "@/components/AssetInfoModal.vue";
import { ACTIONS } from "@/constants/store";

export default defineComponent({
  name: "AssetsView",
  components: {
    EmptyLogo,
    AssetInfoModal
  },
  computed: {
    ergPrice(): number {
      return this.$store.state.ergPrice;
    },
    conversionCurrency(): string {
      return this.$store.state.settings.conversionCurrency;
    },
    loading(): boolean {
      if (!this.$store.state.loading.balance) {
        return false;
      }

      const assetList: StateAsset[] = this.$store.getters[GETTERS.NON_PICTURE_NFT_BALANCE];
      if (assetList.length === 0) {
        return true;
      }

      return false;
    },
    assets(): StateAsset[] {
      const assetList = this.$store.getters[GETTERS.NON_PICTURE_NFT_BALANCE];

      if (this.filter !== "" && assetList.length > 0) {
        return assetList.filter((a: StateAsset) =>
          a.info?.name?.toLocaleLowerCase().includes(this.filter.toLocaleLowerCase())
        );
      }

      return assetList;
    },
    hideBalances(): boolean {
      return this.$store.state.settings.hideBalances;
    }
  },
  watch: {
    ["assets.length"](newLen, oldLen) {
      const length = oldLen || 1;
      if (length > 1) {
        this.prevCount = length;
      }
    }
  },
  data() {
    return {
      filter: "",
      prevCount: 1,
      selectedAsset: undefined as StateAsset | undefined
    };
  },
  methods: {
    price(tokenId: string): BigNumber {
      const rate = this.rate(tokenId);
      if (!rate || !this.ergPrice) {
        return new BigNumber(0);
      }

      return new BigNumber(rate).multipliedBy(this.ergPrice);
    },
    rate(tokenId: string): number {
      return this.$store.state.assetMarketRates[tokenId]?.erg ?? 0;
    },
    isErg(tokenId: string): boolean {
      return tokenId === ERG_TOKEN_ID;
    },
    toggleHideBalance(): void {
      this.$store.dispatch(ACTIONS.TOGGLE_HIDE_BALANCES);
    }
  }
});
</script>
