{#if $active}
  <div class="boxgpt-root">
    <div class="boxgpt-header">
      <div class="boxgpt-btn-close" on:click={closeBox}>ⓧ</div>
      <div class="space"></div>
      <div class="boxgpt-text">Ctrl + \ : ON/OFF</div>
    </div>
    <div class="boxgpt-conversation">
      <div class="boxgpt-messages">
        {#each messages as message}
          {#if message.type === 'user'}
            <div class="boxgpt-message boxgpt-message-user">
              <div class="boxgpt-message-text">
                <div class="boxgpt-message-indicator">▶</div>
                {renderMessage(message)}
              </div>
            </div>
          {:else}
            <div class="boxgpt-message boxgpt-message-gpt">
              <div class="boxgpt-message-text">{renderMessage(message)}</div>
            </div>
          {/if}
        {:else}
          <div class="boxgpt-message boxgpt-message-gpt">
            <div class="boxgpt-message-text">Hello! Ask me anything.</div>
          </div>
        {/each}
      </div>
    </div>
    <div class="boxgpt-prompt">
      <div class="boxgpt-message-indicator">▶</div>
      <textarea disabled={$authorized !== 'authorized'} bind:value={prompt} on:keypress={handleTextInput}></textarea>
      {#if $authorized === 'unauthorized'}
        <div class="boxgpt-message-login" on:click={doLogin}>Log In</div>
      {/if}
    </div>
  </div>
{/if}

<script lang="ts">
  import {writable, Writable} from "svelte/store";

  const port = chrome?.runtime?.connect();

  let authorized = writable(''); // authorized, unauthorized, or empty string
  let active = writable(null); // true, false, or null
  let prompt = '';
  let messages = [];

  $: {
    if ($active) {
      document.documentElement.classList.add('boxgpt-active');
    } else {
      document.documentElement.classList.remove('boxgpt-active');
    }
    if ($active !== null) {
      port.postMessage({type: 'meta', action: 'set-active', active: $active});
    }
  }
  $: {
    console.debug('authorized', $authorized);
  }

  setup();

  function setup() {
    document.documentElement.addEventListener('keydown', (event) => {
      if (event.ctrlKey && event.key === '\\') {
        toggleBox();
      }
    });
    if (sessionStorage.getItem('boxgpt-active') === 'true') {
      $active = true;
    }

    if (!port) {
      return;
    }
    port.onMessage.addListener((msg) => {
      console.debug('received message', msg);
      if (msg.type === 'meta') {
        if (msg.status === 'authorized' || msg.status === 'unauthorized') {
          $authorized = msg.status;
        }
        if (msg.action === 'toggle') {
          toggleBox();
        }
        if (msg.action === 'set-active') {
          $active = msg.active;
        }
        return;
      }
      if (msg.message) {
        appendMessage(msg);
      } else {
        console.debug('discard msg', msg);
      }
    });
  }

  function toggleBox() {
    $active = !$active;
    if ($active) {
      setTimeout(() => {
        document.querySelector('.boxgpt-prompt textarea')?.focus();
      }, 10);
    }
  }

  function closeBox() {
    $active = false;
  }

  function doLogin() {
    port.postMessage({type: 'meta', action: 'login'});
  }

  function handleTextInput(event) {
    if (event.key === 'Enter' && !event.ctrlKey && !event.metaKey && !event.shiftKey && !event.altKey) {
      event.preventDefault();

      postQuestion({question: prompt});
      prompt = '';
    }
  }

  function postQuestion(data) {
    if (port) {
      port.postMessage(data);
    } else {
      console.debug('post question', data);
    }
  }

  function renderMessage(msg) {
    return msg?.message?.content?.parts?.[0] || '';
  }

  function appendMessage(message) {
    if (messages.at(-1)?.status === 'thinking') {
      messages.pop();
    }

    if (message.message?.id) {
      for (let i = messages.length - 1; i >= 0; i--) {
        if (messages[i].message?.id === message.message?.id) {
          messages[i] = message;
          messages = messages;
          return;
        }
      }
    }
    messages.push(message);
    messages = messages.slice(-40);
  }
</script>

<style lang="scss">
  .boxgpt-root {
    position: fixed;
    right: 0;
    top: 0;
    bottom: 0;
    width: 400px;
    background: #343540;
    display: flex;
    flex-direction: column;
    z-index: 1000000000;
  }

  .boxgpt-header {
    height: 30px;
    background: #2b2b36;
    display: flex;
    align-items: center;
    padding: 0 16px;
    color: #d2d5da;
  }

  .space {
    flex: 1;
  }

  .boxgpt-header .boxgpt-btn-close {
    width: 30px;
    height: 30px;
    line-height: 30px;
    cursor: pointer;
  }

  .boxgpt-header .boxgpt-text {
    opacity: 60%;
  }

  .boxgpt-conversation {
    flex: 1 0 0;
    overflow-y: auto;
  }

  .boxgpt-messages {
    padding: 20px 0;
    color: #d2d5da;
    font-size: 14px;
    line-height: 20px;
  }

  .boxgpt-message {
    padding: 20px 16px;
    position: relative;
  }

  .boxgpt-message-text {
    position: relative;
  }

  .boxgpt-message-indicator {
    position: absolute;
    top: 0;
    left: -15px;
    width: 4px;
    font-size: 8px;
    color: #d2d5da;
    opacity: 60%;
    user-select: none;
  }

  .boxgpt-message-user {
    background: #40414e;
  }

  .boxgpt-prompt {
    flex: 0 0 auto;
    background: #40414e;
    padding: 3px 6px;
    position: relative;
  }

  .boxgpt-prompt .boxgpt-message-indicator {
    top: 12px;
    left: 1px;
    color: #d2d5da;
    opacity: 60%;
  }

  .boxgpt-prompt textarea {
    display: block;
    width: 100%;
    height: 80px;
    padding: 5px 10px;
    color: #fff;
    background: #40414e;
    outline: none;
    border: none;
  }

  .boxgpt-prompt textarea:focus {
    outline: none;
  }

  .boxgpt-message-login {
    background: #4aa181;
    border-radius: 4px;
    padding: 6px 10px;
    position: absolute;
    top: 4px;
    cursor: pointer;
  }

  .boxgpt-message-login:hover {
    background: #3f8a6f;
  }
</style>
