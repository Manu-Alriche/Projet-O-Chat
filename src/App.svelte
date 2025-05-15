<!-- JAVASCRIPT ------------------------------------------------------------------------------------- -->
<script>
  import { onMount } from 'svelte';
  const token = import.meta.env.VITE_MISTRAL_API_TOKEN;
  
  let prompt = $state("");
  let messages = $state([]);
  let conversations = $state([]);
  let currentConversationId = $state(null);
  let newConversationTitle = $state('');
  let showSidebar = $state(false);

  // Charger les conversations existantes
  async function loadConversations() {
    const res = await fetch('http://127.0.0.1:8090/api/collections/conversations/records');
    const data = await res.json();
    conversations = data.items.sort((a, b) => new Date(b.created) - new Date(a.created));
  }

  // Créer une nouvelle conversation
  async function createConversation() {
    if (!newConversationTitle.trim()) return;

    const res = await fetch('http://127.0.0.1:8090/api/collections/conversations/records', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: newConversationTitle })
    });

    const newConv = await res.json();
    conversations = [newConv, ...conversations];
    newConversationTitle = '';
  }

  // Charger les messages d'une conversation
  async function loadMessages(conversationId) {
    currentConversationId = conversationId;
    const res = await fetch(`http://127.0.0.1:8090/api/collections/messages/records?filter=conversation="${conversationId}"`);
    const data = await res.json();
    messages = data.items
      .sort((a, b) => new Date(a.created) - new Date(b.created))
      .map(msg => ({
        ...msg,
        role: msg.is_ai_response ? 'assistant' : 'user'
      }));
  }

 // Supprimer une conversation et ses messages seulement si l’utilisateur le veut
 async function deleteConversation(conversationId) {
    if (!confirm("⚠️ Voulez-vous vraiment supprimer cette conversation et tous ses messages ?")) return;

    try {
      // 1️⃣ Supprimer tous les messages liés à cette conversation
      const res = await fetch(`http://127.0.0.1:8090/api/collections/messages/records?filter=conversation="${conversationId}"`);
      const data = await res.json();

      for (const msg of data.items) {
        await fetch(`http://127.0.0.1:8090/api/collections/messages/records/${msg.id}`, {
          method: 'DELETE'
        });
      }

      // 2️⃣ Supprimer la conversation elle-même
      await fetch(`http://127.0.0.1:8090/api/collections/conversations/records/${conversationId}`, {
        method: 'DELETE'
      });

      // 3️⃣ Mettre à jour l’affichage : retirer la conversation de la liste locale
      conversations = conversations.filter(conv => conv.id !== conversationId);

      // 4️⃣ Si on supprimait la conversation active, on vide l’écran de chat
      if (currentConversationId === conversationId) {
        currentConversationId = null;
        messages = [];
      }

    } catch (error) {
      console.error('Erreur lors de la suppression:', error);
      alert("❌ La suppression a échoué.");
    }
  }

  
  async function callMistral(){
     // Afficher immédiatement dans le chat
     messages = [...messages, { role: "user", content: prompt, is_ai_response: false }];

    // Enregistre le message utilisateur dans Pocketbase
    await fetch("http://127.0.0.1:8090/api/collections/messages/records", {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        content: prompt,
        is_ai_response: false,
        conversation: currentConversationId,
      })
    });
    
    // Appel à l'API Mistral
    const responseText = await fetch(
        "https://api.mistral.ai/v1/chat/completions",
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            "model": "mistral-small-latest",
            "messages": [{
              "role": "user",
              "content": prompt}]
          })         
        }
      );
    const data = await responseText.json();
    console.log(data);
    const botResponse = data.choices?.[0]?.message?.content ?? "Pas de reponse.";
    
    // Afficher la réponse dans le chat
    messages = [...messages, { role: "assistant", content: botResponse, is_ai_response: true }];

    // Stocker la réponse IA
    await fetch("http://127.0.0.1:8090/api/collections/messages/records", {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      content: botResponse,
      is_ai_response: true,
      conversation: currentConversationId,
    })
  });

  // Vide le champ texte
  prompt = "";
  }

   // Chargement initial
   onMount(() => {
    loadConversations();
  });

</script>

<!-- HTML ------------------------------------------------------------------------------------------- -->
<div class="desktop">
  <button class="burger" onclick={() => showSidebar = !showSidebar}>
    {#if showSidebar}
        ✖
    {:else}
        ☰
    {/if}
  </button>
  <div class="sidebar" class:open={showSidebar}>   
        <div class="blockA">
          <h1>O'Chat</h1>
            <h2>CONVERSATIONS</h2>
            <ul class="chat-list">
              {#each conversations as conv}
                <li class="chat-item" onclick={() => loadMessages(conv.id)} class:active={conv.id === currentConversationId}>
                  {conv.title} 
                  <button class="close" onclick={() => deleteConversation(conv.id)}>X</button>
                </li>
              {/each}
            </ul>
        </div>
        <div class= "blockB">
            <input bind:value={newConversationTitle} type="text" placeholder="Nouveau titre de conversation..." />
            <button class="create" type="submit" onclick={createConversation}>Créer ➕</button>
        </div>
      </div>
      <div class="chat-window">
          <div class= "blockC" >
            {#each messages as message}
            <div class="message {message.role}">
              <div class="label">{message.role === 'assistant' ? '🤖 IA' : '🧑 Utilisateur'}</div>
              <div class="bubble">{message.content}</div>
            </div>
            <div bind:this={scrollAnchor}></div>
          {/each}
          </div>
          <div class= "blockD">
              <textarea bind:value={prompt} placeholder="Tapez votre message..." cols="200" rows="7" 
               onkeydown={(e) => {
                if (e.key === "Enter" && !e.shiftKey) {
                  e.preventDefault();
                  callMistral();
                }
              }}></textarea>
              <button class="btn" type="submit" onclick={callMistral}>Envoyer</button>
          </div>
      </div>
</div>

<!-- CSS -------------------------------------------------------------------------------------------- -->
<style>
/* Desktop */
:global(body, html) {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: Arial, Helvetica, sans-serif;
  height: 100%;
}

:global(#svelte) {
  height: 100%;
}
.desktop {
  display: flex;
  height: 100vh;
  width: 100%;
}

.burger {
  display: none;
  position: fixed; 
  top: 10px;
  font-size: 2rem;
  padding: 0.5rem;
  background: none;
  border: none;
  cursor: pointer;
  z-index: 1100;
}

/* Sidebar */
.sidebar {
  background: #5E5E5E;
  width: 250px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 15px;
  color: white;
  border-right: 3px solid #000;
  transition: transform 0.3s ease;
}

.sidebar h1 { 
  text-align: center;
}

.sidebar h2 {
  text-align: center;
  font-size: 1.2rem;
  margin-bottom: 30px;
}

.chat-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}
.chat-item {
  display: flex;
  justify-content: space-between;
  background: #555;
  padding: 10px;
  border: 1px solid black;
  border-radius: 0.5rem;
  box-shadow: 2px 1px 1px black;
  align-items: center;
  cursor: pointer;
}

.chat-item.active {
    background: #aaaaaa;
    font-weight: bold;
}

.chat-item .close {
  background: #6c63ff;
  color: white;
  border: none;
  padding: 5px 10px;
  cursor: pointer;
  border-radius: 0.5rem;
  transition: opacity 0.2s;
}

ul {
  padding-inline-start: 0;
}

.blockB {
    display: flex;
    flex-direction: column;
    gap: 15px;
}
.sidebar input {
  padding: 15px;
  box-shadow: 2px 1px 1px black;
  border-radius: 0.5rem;
}

.create {
  padding: 0.75rem 1.5rem;
  background-color: #6c63ff;
  box-shadow: 2px 1px 1px black;
  color: white;
  cursor: pointer;
  border: none;
  border-radius: 0.5rem;
  transition: opacity 0.2s;
}

/* Chat area */
.chat-window {
  flex: 1;
  background: #fff;
  padding: 20px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 15px;
  color: #000;
}

.blockC {
  max-height: 80vh;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  gap: 15px;
}

.user {
    align-self: flex-start;
    text-align: left;
  }

  .assistant {
    align-self: flex-end;
    text-align: right;
  }

  .bubble {
    padding: 0.7rem 1rem;
    border-radius: 1rem 1rem 1rem 0;
    background-color: #c6e7ff;
    display: inline-block;
  }

  .assistant .bubble {
    background-color: #ffd9a0;
    border-radius: 1rem 1rem 0 1rem ;
  }

  .label {
    font-size: 0.75rem;
    margin-bottom: 0.2rem;
    color: #777;
  }
  
.blockD {
    display: flex;
    justify-content: space-around;
    align-items: center;
    background-color: #555;
    box-shadow: 2px 1px 1px black;
    border-radius: 1rem;
    padding: 10px;
    margin: 10px;
}

textarea {
  flex: 1;
    padding: 0.75rem;
    border: 1px solid #dee2e6;
    border-radius: 0.5rem;
    resize: none;
    box-shadow: 2px 1px 1px black;
}

.btn {
  padding: 0.75rem 1.5rem;
  margin: 10px;
  background-color: #6c63ff;
  color: white;
  border: none;
  box-shadow: 2px 1px 1px black;
  cursor: pointer;
  border-radius: 0.5rem;
  transition: opacity 0.2s; 
}

/* Mobile */
@media (max-width: 768px) {
  .desktop {
    flex-direction: column;
    height: auto;
  }

  .burger {
    display: block;
    position: fixed;
    top: 1px;
    right: 10px; 
    font-size: 2rem;
    padding: 0.5rem;
    background: none;
    border: none;
    cursor: pointer;
    z-index: 1200;
  }

  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    max-height: 80%; 
    background: #5E5E5E;
    transform: translateY(-100%);
    transition: transform 0.3s ease;
    z-index: 1100;
    padding: 20px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.3);
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    gap: 15px;
    overflow-y: auto;
  }
  .sidebar.open {
    transform: translateY(0);
  }
  .blockA,
  .blockB {
    width: 95%;
  }

  .chat-item {
    font-size: 0.9rem;
  }

  .chat-window {
    padding: 10px;
    gap: 10px;
    overflow-y: auto;
  }

  .blockC {
    max-height: 80vh;
    overflow-y: auto;
    gap: 10px;
    margin-top: 2rem;
  }

  .bubble {
    max-width: 90%;
    font-size: 0.95rem;
    word-wrap: break-word;
  }

  .bubble.right {
    align-self: flex-end;
    background: #c0c0ff;
  }

  .blockD {
    position: fixed;
    bottom: 0;
  }

  textarea {
    width: 95%;
    max-width: 80%;
    font-size: 0.7rem;
  }

  .btn {
    padding: 0.75rem 1.5rem;
    font-size: 0.8rem;
    background-color: #6c63ff;
    color: white;
    border: none;
    cursor: pointer;
    transition: opacity 0.2s;
  }
}
</style>