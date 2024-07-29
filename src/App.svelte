<svelte:options runes={true} />
<script>
  // @ts-check
  import Svelecte from 'svelecte';
  // import { dummyData } from "./dummyData"

  const apiBase = "https://api.coincap.io/v2";

  /**
   * @typedef {`{number}`} NumberStringWhole
   */
  /**
   * @typedef {`{number}.{number}`} NumberStringDecimal
   */
  /**
   * @typedef {NumberStringDecimal | NumberStringWhole} NumberString
   */
  /**
   * @typedef {{
   *   "id": string,
  //  *   "rank": NumberStringDecimal,
   *   "symbol": string,
   *   "name": string,
  //  *   "supply": NumberString,
  //  *   "maxSupply": null | string,
  //  *   "marketCapUsd": NumberString,
  //  *   "volumeUsd24Hr": NumberString,
   *   "priceUsd": NumberString,
  //  *   "changePercent24Hr": `{number}.{number}` | `-{number}.{number}`,
  //  *   "vwap24Hr": NumberString,
   * }} Asset
   */

  /** @type {Map<Asset["id"], Asset> | null} */
  let assetsMap = $state(null);
  /** @type {Asset[] | null} */
  let assetsArray = $state(null);

  // loadAssetsCache
  (typeof chrome !== "undefined"
    ? async () => {
      const { assetsCache } = await chrome.storage.local.get("assetsCache");
      if (assetsCache) {
        assetsArray = assetsCache;
        assetsMap = assetsMapFromAssetsArray(assetsCache);
        console.log("Loaded assetsCache");
      }
    } : async () => {
      const assetsCacheStr = localStorage.getItem("assetsCache");
      if (assetsCacheStr) {
        const assetsCache = JSON.parse(assetsCacheStr);
        assetsArray = assetsCache;
        assetsMap = assetsMapFromAssetsArray(assetsCache);
        console.log("Loaded assetsCache");
      }
    }
  )()

  // TODO perhaps don't use default values while settings are loading.
  // Separating this into a component might help.
  //
  // `null` because with Svelecte it's possible to unselect
  // https://github.com/mskocik/svelecte/issues/172
  /** @type {Asset["id"] | null} */
  let fromAssetId = $state("bitcoin");
  let fromAmount = $state(1);
  /** @type {Asset["id"] | null} */
  let toAssetId = $state("ethereum");

  const loadSettings = typeof chrome !== "undefined"
    ? async () => {
      const settings = await chrome.storage.local.get()
      fromAssetId = settings.fromAssetId ?? fromAssetId;
      fromAmount = settings.fromAmount ?? fromAmount;
      toAssetId = settings.toAssetId ?? toAssetId;
    } : async () => {
      fromAssetId = localStorage.getItem("fromAssetId") ?? fromAssetId;
      fromAmount = localStorage.getItem("fromAmount")
        ? parseFloat(localStorage.getItem("fromAmount"))
        : fromAmount;
      toAssetId = localStorage.getItem("toAssetId") ?? toAssetId;
    }

  loadSettings();

  // TODO import types for Chrome.
  const saveSettings = typeof chrome !== "undefined"
    ? () => {
      chrome.storage.local.set({
        fromAssetId,
        fromAmount,
        toAssetId,
      })
    } : () => {
      localStorage.setItem("fromAssetId", fromAssetId);
      localStorage.setItem("fromAmount", fromAmount.toString());
      localStorage.setItem("toAssetId", toAssetId);
    }

  $effect(() => {
    console.log("Saving settings");
    saveSettings();
  })

  const fetchAssetsP = (async function fetchAssets() {
    // TODO this is paginated:
    // https://docs.coincap.io/#89deffa0-ab03-4e0a-8d92-637a857d2c91
    // we should fetch another 2000, if they do exist.
    //
    // TODO perf: cache this, at least just the descriptions of the assets,
    // otherwise the user can't interact with the UI
    // until this has loaded.
    const res = await fetch(`${apiBase}/assets?limit=2000`);
    const assetsArray_ = (await res.json()).data.map(asset => ({
      // Only use the things that we need for performance,
      // especially for caching.
      id: asset.id,
      symbol: asset.symbol,
      name: asset.name,
      priceUsd: asset.priceUsd,
    }))
    // assetsArray = (await res.json()).data;
    // const assetsArray = (dummyData).data;
    assetsArray_.unshift({
      id: "_usd",
      name: "USD",
      symbol: "USD",
      priceUsd: /** @type {NumberString} */ ("1"),
    })

    assetsArray = assetsArray_;
    assetsMap = assetsMapFromAssetsArray(assetsArray_);

    // storeAssetsCache
    // TODO perf: It would probably be better and more maintainable
    // to just store the API reponse, using a ServiceWorker or something.
    // Loading from storage takes a while.
    (typeof chrome !== "undefined"
      ? (assetsArray) => {
        console.log("Saving assetsCache");
        chrome.storage.local.set({
          // Looks like Svelte's proxy thing stores this as an Object
          // instead of an array.
          assetsCache: Array.from(assetsArray),
        })
      } : (assetsArray) => {
        console.log("Saving assetsCache");
        localStorage.setItem("assetsCache", JSON.stringify(assetsArray));
      }
    )(assetsArray_)

    // TODO subscribe to price updates (see API docs).
  })();

  /**
   * @param {Asset[]} assetsArray 
   * @returns {Map<Asset["id"], Asset>}
   */
  function assetsMapFromAssetsArray(assetsArray) {
    const assetsMap_ = new Map();
    for (const asset of assetsArray) {
      assetsMap_.set(asset.id, asset);
    }
    return assetsMap_;
  }
</script>

<main>
  {#if assetsArray && assetsMap}
    <form style="display: grid;">
      <!-- TODO add step? -->
      <!-- TODO fix: sometimes, if you save a really low amount,
      it restores it in a scientific notation... -->
      <input
        type="number"
        bind:value={fromAmount}
        min="0"
        style="grid-column: 1; grid-row: 1;"
        aria-label="From amount"
      />

      <!-- TODO aria-label="From currency" -->
      <!-- TODO sort them alphabetically? Same for another Svelecte -->
      <!-- TODO style: it expands the popup, unlike a native "select" -->
      <!-- TODO add other fiat currencies, use `/rates` endpoint -->
      <div
        style="grid-column: 1; grid-row: 2;"
      >
        <Svelecte
          bind:value={fromAssetId}
          options={assetsArray}
          valueField={'id'}
          labelField={'name'}

          name="fromAssetId"
          required={true}
        />
      </div>

      <p
        style="
          grid-column: 2;
          grid-row: 1;
          text-align: center;
          margin: 0 0.5rem;
        "
      >=</p>

      <!-- TODO aria-label="To currency" -->
      <div
        style="grid-column: 3; grid-row: 2;"
      >
        <Svelecte
          bind:value={toAssetId}
          options={assetsArray}
          valueField={'id'}
          labelField={'name'}

          name="toAssetId"
          required={true}
        />
      </div>

      <!-- Yes, it's OK to put `output` inside of a <form> -->
      <output
        style="grid-column: 3; grid-row: 1;"
      >{
        // TODO perhaps simply dividing is not great?
        // IDK, precise enough I think.
        //
        // TODO we'll probably need better formatting, too much precision sometimes
        (
          fromAssetId
          && toAssetId
        ) ? (
          parseFloat(assetsMap.get(fromAssetId).priceUsd)
          / parseFloat(assetsMap.get(toAssetId).priceUsd)
          * fromAmount
        ).toFixed(20)
        : "-"
      }</output>

      <button
        type="button"
        aria-label="Swap"
        onclick={() => {
          [fromAssetId, toAssetId] = [toAssetId, fromAssetId];
        }}
        style="
          justify-self: center;
        "
      >🔄</button>
    </form>
  {/if}
  {#await fetchAssetsP}
    <p>Loading...</p>
  {:then}
  <!-- svelte-ignore block_empty -->
  {:catch}
    <p>Failed to fetch data 😖</p>
  {/await}
</main>

<style>
@media (prefers-color-scheme: dark) {
  /* Svelecte theme
  https://svelecte-v5.vercel.app/theme
  Copy-pasted from the website. */
  :root {
    --sv-min-height: 40px;
    --sv-bg: #383838;
    --sv-disabled-bg: #222;
    --sv-border: 1px solid #626262;
    --sv-border-radius: 4px;
    --sv-general-padding: 4px;
    --sv-control-bg: var(--sv-bg);
    --sv-item-wrap-padding: 3px 3px 3px 6px;
    --sv-selection-wrap-padding: 3px 3px 3px 4px;
    --sv-selection-multi-wrap-padding: 3px 3px 3px 6px;
    --sv-item-selected-bg: #5c73e7;
    --sv-item-btn-color: #ccc;
    --sv-item-btn-color-hover: #ccc;
    --sv-item-btn-bg: #626262;
    --sv-item-btn-bg-hover: #3a5ccc;
    --sv-icon-color: #bbb;
    --sv-icon-color-hover: #ccc;
    --sv-icon-bg: transparent;
    --sv-icon-size: 20px;
    --sv-separator-bg: #626262;
    --sv-btn-border: 0;
    --sv-placeholder-color: #ccccd6;
    --sv-dropdown-bg: var(--sv-bg);
    --sv-dropdown-border: var(--sv-border);
    --sv-dropdown-offset: 1px;
    --sv-dropdown-width: auto;
    --sv-dropdown-shadow: 0 1px 3px #555;
    --sv-dropdown-height: 320px;
    --sv-dropdown-active-bg: #555555;
    --sv-dropdown-selected-bg: #754545;
    --sv-create-kbd-border: 1px solid #626262;
    --sv-create-kbd-bg: #626262;
    --sv-create-disabled-bg: #fcbaba;
    --sv-loader-border: 2px solid #626262
  }
}
</style>
