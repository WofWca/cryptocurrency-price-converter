<svelte:options runes={true} />
<script>
  // @ts-check
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
  let assets = $state(null);

  // TODO perhaps don't use default values while settings are loading.
  // Separating this into a component might help.
  let fromAssetId = $state("bitcoin");
  let fromAmount = $state(1);
  let toAssetId = $state("ethereum");

  const loadSettings = typeof chrome !== "undefined"
    ? async () => {
      // TODO fuck, they might be `undefined`.
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

  const initP = (async function fetchAssets() {
    // TODO this is paginated:
    // https://docs.coincap.io/#89deffa0-ab03-4e0a-8d92-637a857d2c91
    // we should fetch another 2000, if they do exist.
    //
    // TODO perf: cache this, at least just the descriptions of the assets,
    // otherwise the user can't interact with the UI
    // until this has loaded.
    const res = await fetch(`${apiBase}/assets?limit=2000`);
    const assetsArray = (await res.json()).data;
    // const assetsArray = (dummyData).data;

    const assetsMap = new Map();

    /** @type {Asset} */
    const usdDummyAsset = {
      id: "_usd",
      name: "USD",
      symbol: "USD",
      priceUsd: /** @type {NumberString} */ ("1"),
    }
    assetsMap.set(usdDummyAsset.id, usdDummyAsset)

    for (const asset of assetsArray) {
      assetsMap.set(asset.id, asset);
    }
    assets = assetsMap

    // TODO subscribe to price updates (see API docs).
  })();
</script>

<main>
  {#await initP}
    <p>Loading...</p>
  {:then}
    {#snippet selectOptions()}
      <!-- TODO add other fiat currencies, use `/rates` endpoint -->
      {#each assets as [_, asset] (asset.id)}
        <option value={asset.id}>{asset.symbol}</option>
      {/each}
    {/snippet}
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

      <!-- TODO proper search instead of select? -->
      <!-- TODO sort them alphabetically... -->
      <select
        bind:value={fromAssetId}
        style="grid-column: 1; grid-row: 2;"
        aria-label="From currency"
      >
        {@render selectOptions()}
      </select>

      <p
        style="
          grid-column: 2;
          grid-row: 1;
          text-align: center;
          margin: 0 0.5rem;
        "
      >=</p>

      <select
        bind:value={toAssetId}
        style="grid-column: 3; grid-row: 2;"
        aria-label="To currency"
      >
        {@render selectOptions()}
      </select>

      <!-- Yes, it's OK to put `output` inside of a <form> -->
      <output
        style="grid-column: 3; grid-row: 1;"
      >{
        // TODO perhaps simply dividing is not great?
        // IDK, precise enough I think.
        //
        // TODO we'll probably need better formatting, too much precision sometimes
        (
          parseFloat(assets.get(fromAssetId).priceUsd)
          / parseFloat(assets.get(toAssetId).priceUsd)
          * fromAmount
        ).toFixed(20)
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
  {:catch}
    <p>Failed to fetch data 😖</p>
  {/await}
</main>

<style>
</style>
