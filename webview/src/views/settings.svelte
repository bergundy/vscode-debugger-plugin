<script lang="ts">
  import { onDestroy } from "svelte"
  import type { ViewSettings } from "../lib"

  export let eventEmitter: EventTarget
  const settingsLoadedPromise = new Promise<ViewSettings>((resolve) => {
    const listener = (e) => {
      resolve((e as CustomEvent<ViewSettings>).detail)
    }
    eventEmitter.addEventListener("settingsLoaded", listener, { once: true })
    onDestroy(() => eventEmitter.removeEventListener("settingsLoaded", listener))
  })

  /**
   * Event listener for saving the settings
   */
  async function saveSettings(e: Event) {
    // TODO: handle errors
    if (!(e.target instanceof HTMLFormElement)) {
      throw new TypeError("Expected form element")
    }

    const data = new FormData(e.target)

    const settings = Object.fromEntries(
      await Promise.all(
        [["tls", "off"], ...data.entries()].map(async ([k, v]) => {
          if (k === "clientCert" || k === "clientPrivateKey") {
            const fileblob = new Blob([v])
            const buffer = await fileblob.arrayBuffer()
            return [k, buffer]
          }
          if (k === "tls") {
            return [k, v === "on"]
          }
          return [k, v]
        }),
      ),
    )

    vscode.postMessage({
      type: "updateSettings",
      settings,
    })
  }

  // Load settings on component load
  vscode.postMessage({ type: "getSettings" })
</script>

<section>
  {#await settingsLoadedPromise}
    Loading...
  {:then settings}
    <p>Configure client connection (for downloading histories)</p>
    <form on:submit|preventDefault={saveSettings}>
      <label for="address">Address</label>
      <input type="text" required value={settings.address} name="address" />
      <label for="tls">TLS?</label>
      <input type="checkbox" name="tls" checked={settings.tls} />
      <div />
      <label for="clientCert">Client cert {settings.hasClientCert ? "(present)" : ""}</label>
      <input type="file" name="clientCert" />
      <div />
      <label for="clientPrivateKey">Client private key {settings.hasClientPrivateKey ? "(present)" : ""}</label>
      <input type="file" name="clientPrivateKey" />
      <div />
      <button type="submit">Submit</button>
    </form>
  {/await}
</section>
