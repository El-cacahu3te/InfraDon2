<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

interface Post {
  _id: string
  _rev?: string
  post_name: string
  post_content: string
  total_likes?: number
  comments?: string[]
  attributes?: {
    creation_date: any
  }
  _conflicts?: string[]
}

interface Reaction {
  _id: string
  _rev?: string
  user_id: string
  post_id: string
  isliked: boolean
  comments: string[]
}

// Variables réactives
const documentName = ref('')
const documentContent = ref('')
const postsData = ref<Post[]>([])
const showSyncMessage = ref(false)
const isOffline = ref(false)
const selectedConflict = ref<Post | null>(null)
const otherVersions = ref<Post[]>([])
const buttonName = ref('En ligne')
const searchTerm = ref('')
const newComment = ref('')
const reactionsData = ref<Reaction[]>([])

const postsDB = ref<PouchDB.Database<Post>>()
const reactionsDB = ref<PouchDB.Database<Reaction>>()

let syncHandler: any = null
let postsSyncHandler: any = null
let reactionsSyncHandler: any = null


// ==================== INITIALISATION ====================
const initDatabase = async () => {
  console.log('🔧 Initialisation des bases...')
  
  // Créer les bases locales
  postsDB.value = new PouchDB<Post>('local_posts')
  reactionsDB.value = new PouchDB<Reaction>('local_reactions')
  console.log('✅ Bases locales créées')

  // Credentials en Base64 pour l'authentification
  const username = 'elia'
  const password = 'admin'
  const authHeader = 'Basic ' + btoa(username + ':' + password)

  // URLs CouchDB SANS credentials
  const remotePostsURL = 'http://localhost:5984/post_elia_nicolo'
  const remoteReactionsURL = 'http://localhost:5984/reactions_elia_nicolo'

  // TEST 1 : CouchDB répond ?
  console.log('🔍 Test connexion CouchDB...')
  try {
    const response = await fetch('http://localhost:5984')
    const data = await response.json()
    console.log('✅ CouchDB répond:', data)
  } catch (err) {
    console.error('❌ CouchDB ne répond pas:', err)
    return
  }

  // TEST 2 : Les bases existent ?
  console.log('🔍 Vérification des bases...')
  try {
    const response = await fetch('http://localhost:5984/_all_dbs', {
      headers: {
        'Authorization': authHeader
      }
    })
    const dbs = await response.json()
    console.log('📦 Bases disponibles:', dbs)
    
    if (!dbs.includes('post_elia_nicolo')) {
      console.error('❌ La base post_elia_nicolo n\'existe pas')
      return
    }
    if (!dbs.includes('reactions_elia_nicolo')) {
      console.error('❌ La base reactions_elia_nicolo n\'existe pas')
      return
    }
    console.log('✅ Les deux bases existent')
  } catch (err) {
    console.error('❌ Erreur auth/bases:', err)
    return
  }

  // TEST 3 : Combien de docs dans CouchDB ?
  console.log('🔍 Comptage des documents dans CouchDB...')
  try {
    const postsResponse = await fetch('http://localhost:5984/post_elia_nicolo/_all_docs', {
      headers: { 'Authorization': authHeader }
    })
    const postsData = await postsResponse.json()
    console.log(`📄 ${postsData.total_rows} documents dans post_elia_nicolo`)
    console.log('Détails:', postsData.rows)
    
    const reactionsResponse = await fetch('http://localhost:5984/reactions_elia_nicolo/_all_docs', {
      headers: { 'Authorization': authHeader }
    })
    const reactionsData = await reactionsResponse.json()
    console.log(`💬 ${reactionsData.total_rows} documents dans reactions_elia_nicolo`)
  } catch (err) {
    console.error('❌ Erreur comptage docs:', err)
  }

  // Créer les index
  try {
    await postsDB.value.createIndex({
      index: { fields: ['post_name'] }
    })
    await postsDB.value.createIndex({
      index: { fields: ['total_likes'] }
    })
    await reactionsDB.value.createIndex({
      index: { fields: ['post_id', 'user_id'] }
    })
    console.log('✅ Index créés')
  } catch (err) {
    console.error('❌ Erreur index:', err)
  }

  // Sync initiale one-shot pour charger les données
console.log('🔄 Synchronisation initiale...')
try {
  await postsDB.value.sync('http://elia:admin@localhost:5984/post_elia_nicolo')
  console.log('✅ Posts synchronisés')
  
  await reactionsDB.value.sync('http://elia:admin@localhost:5984/reactions_elia_nicolo')
  console.log('✅ Reactions synchronisées')
} catch (err) {
  console.error('❌ Erreur sync initiale:', err)
}


  // Charger les données dans l'UI
  console.log('🎨 Chargement des données dans l\'interface...')
  await fetchData()

  
 
}

// ==================== SYNCHRONISATION LIVE ====================
const startLiveSync = () => {
  if (!postsDB.value || !reactionsDB.value) return

  console.log('🔄 Démarrage sync bidirectionnelle...')

  const remotePostsDB = new PouchDB('http://elia:admin@localhost:5984/post_elia_nicolo')
  const remoteReactionsDB = new PouchDB('http://elia:admin@localhost:5984/reactions_elia_nicolo')

  // Sync POSTS bidirectionnel AVEC authentification
  postsSyncHandler = postsDB.value.sync(remotePostsDB, {
    live: true,
    retry: true
  })
    .on('change', (info) => {
      console.log('📦 Changement posts:', info.direction, 'docs:', info.change.docs.length)
      fetchData()
    })
    .on('paused', () => {
      console.log('⏸️ Sync posts en pause (normal)')
    })
    .on('active', () => {
      console.log('▶️ Sync posts reprise')
    })
    .on('error', (err) => {
      console.error('❌ Erreur sync posts:', err)
    })

  // Sync REACTIONS bidirectionnel AVEC authentification
  reactionsSyncHandler = reactionsDB.value.sync(remoteReactionsDB, {
    live: true,
    retry: true
  })
    .on('change', (info) => {
      console.log('💬 Changement reactions:', info.direction, 'docs:', info.change.docs.length)
      fetchData()
    })
    .on('paused', () => {
      console.log('⏸️ Sync reactions en pause')
    })
    .on('active', () => {
      console.log('▶️ Sync reactions reprise')
    })
    .on('error', (err) => {
      console.error('❌ Erreur sync reactions:', err)
    })

  console.log('✅ Sync live activée avec authentification')
}


// ==================== RÉCUPÉRATION DONNÉES ====================
const fetchData = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    // ⭐ AJOUTE conflicts: true pour voir les conflits
    const result = await postsDB.value.allDocs({
      include_docs: true,
      conflicts: true  // ← AJOUTE ÇA
    })

    postsData.value = result.rows
      .filter((row) => row.doc && !row.id.startsWith('_design/'))
      .map((row) => row.doc as Post)
      .sort((a, b) => {
        const dateA = a.attributes?.creation_date || 0
        const dateB = b.attributes?.creation_date || 0
        return dateB - dateA
      })

    // Log les conflits détectés
    const conflicted = postsData.value.filter(p => p._conflicts && p._conflicts.length > 0)
    if (conflicted.length > 0) {
      console.log('⚠️ CONFLITS DÉTECTÉS:', conflicted.map(p => ({
        id: p._id,
        name: p.post_name,
        conflicts: p._conflicts
      })))
    }

    const reactionsResult = await reactionsDB.value.allDocs({
      include_docs: true
    })

    reactionsData.value = reactionsResult.rows
      .filter((row) => row.doc && !row.id.startsWith('_design/'))
      .map((row) => row.doc as Reaction)

    console.log(`📊 ${postsData.value.length} posts, ${reactionsData.value.length} reactions`)
  } catch (err) {
    console.error('❌ Erreur fetchData:', err)
  }
}


// ==================== GESTION CONFLITS ====================
const resolveConflict = async (postId: string) => {
  if (!postsDB.value) return

  try {
    const post = await postsDB.value.get(postId, { conflicts: true })
    
    if (!post._conflicts || post._conflicts.length === 0) {
      console.log('Aucun conflit trouvé')
      return
    }

    selectedConflict.value = post
    otherVersions.value = []

    for (const rev of post._conflicts) {
      const conflictVersion = await postsDB.value.get(postId, { rev })
      otherVersions.value.push(conflictVersion)
    }

    console.log(`⚠️ ${post._conflicts.length} conflit(s) détecté(s)`)
  } catch (err) {
    console.error('❌ Erreur résolution conflit:', err)
  }
}

const keepLocal = async () => {
  if (!selectedConflict.value || !selectedConflict.value._conflicts || !postsDB.value) return

  try {
    for (const rev of selectedConflict.value._conflicts) {
      await postsDB.value.remove(selectedConflict.value._id, rev)
    }

    await fetchData()
    cancelConflictResolution()
    console.log('✅ Version locale conservée')
  } catch (err) {
    console.error('❌ Erreur keepLocal:', err)
  }
}

const keepRemote = async (index: number) => {
  if (!selectedConflict.value || !selectedConflict.value._conflicts || !postsDB.value) return

  try {
    const chosenVersion = otherVersions.value[index] as Post
    
    const newDoc: Post = {
      _id: selectedConflict.value._id,
      _rev: selectedConflict.value._rev,
      post_name: chosenVersion.post_name,
      post_content: chosenVersion.post_content,
      total_likes: chosenVersion.total_likes,
      attributes: chosenVersion.attributes
    }
    
    await postsDB.value.put(newDoc)
    
    for (const rev of selectedConflict.value._conflicts) {
      await postsDB.value.remove(selectedConflict.value._id, rev)
    }
    
    await fetchData()
    cancelConflictResolution()
    console.log('✅ Version distante conservée')
  } catch (err) {
    console.error('❌ Erreur keepRemote:', err)
  }
}

const cancelConflictResolution = () => {
  selectedConflict.value = null
  otherVersions.value = []
}

// ==================== RECHERCHE ====================
const searchPosts = async (term: string) => {
  if (!postsDB.value) return

  if (!term.trim()) {
    await fetchData()
    return
  }

  try {
    const result = await postsDB.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .map(row => row.doc!)
      .filter(doc => 
        !doc._id.startsWith('_design') &&
        doc.post_name.toLowerCase().includes(term.toLowerCase())
      )
  } catch (err) {
    console.error('❌ Erreur recherche:', err)
  }
}

const getTopLikedPosts = async () => {
  if (!postsDB.value) return

  try {
    const result = await postsDB.value.find({
      selector: {
        total_likes: { $gte: 0 }
      },
      sort: [{ total_likes: 'desc' }],
      limit: 10
    })
    postsData.value = result.docs
  } catch (err) {
    console.error('❌ Erreur top likes:', err)
  }
}

// ==================== CRUD POSTS ====================
const addDocument = async () => {
  if (!postsDB.value || !documentName.value.trim()) return

  try {
    // 1. Créer le post en local
    await postsDB.value.post({
      _id: `post_${Date.now()}`,
      post_name: documentName.value,
      post_content: documentContent.value || 'Contenu vide',
      total_likes: 0,
      attributes: {
        creation_date: new Date().toISOString()
      }
    })

    // 2. ⭐ SYNCHRONISER VERS COUCHDB ⭐
    await postsDB.value.replicate.to('http://elia:admin@localhost:5984/post_elia_nicolo')

    documentName.value = ''
    documentContent.value = ''
    await fetchData()
    console.log('✅ Post créé et synchronisé')
  } catch (err) {
    console.error('❌ Erreur ajout document:', err)
  }
}
const updateDocument = async (post: Post, newName: string) => {
  if (!postsDB.value || !post._rev || !newName.trim()) return

  try {
    await postsDB.value.put({
      _id: post._id,
      _rev: post._rev,
      post_name: newName,
      post_content: post.post_content,
      total_likes: post.total_likes,
      attributes: post.attributes
    })

    // ⭐ SYNCHRONISER
    await postsDB.value.replicate.to('http://elia:admin@localhost:5984/post_elia_nicolo')

    await fetchData()
    console.log('✅ Post modifié et synchronisé')
  } catch (err) {
    console.error('❌ Erreur update document:', err)
  }
}

const deleteDocument = async (id: string, rev?: string) => {
  if (!postsDB.value || !rev) return

  try {
    await postsDB.value.remove(id, rev)

    // ⭐ SYNCHRONISER
    await postsDB.value.replicate.to('http://elia:admin@localhost:5984/post_elia_nicolo')

    await fetchData()
    console.log('✅ Post supprimé et synchronisé')
  } catch (err) {
    console.error('❌ Erreur suppression document:', err)
  }
}


// ==================== REACTIONS ====================
const getReactionForPost = (postId: string): Reaction | null => {
  return reactionsData.value.find(
    r => r.post_id === postId && r.user_id === 'user_1'
  ) || null
}
const addReaction = async (postId: string, comment?: string, toggleLike?: boolean) => {
  if (!reactionsDB.value) return

  const reactionId = `reaction_${postId}_user_1`

  try {
    let reaction: Reaction | null = null

    try {
      reaction = await reactionsDB.value.get(reactionId)
    } catch (err: any) {
      if (err.status !== 404) throw err
    }

    if (reaction) {
      let updated = false

      if (toggleLike !== undefined) {
        reaction.isliked = toggleLike
        updated = true
      }

      if (comment && comment.trim()) {
        reaction.comments.push(comment.trim())
        updated = true
      }

      if (updated) {
        await reactionsDB.value.put(reaction)
      }
    } else {
      const newReaction: Reaction = {
        _id: reactionId,
        user_id: 'user_1',
        post_id: postId,
        isliked: toggleLike || false,
        comments: comment && comment.trim() ? [comment.trim()] : []
      }
      await reactionsDB.value.put(newReaction)
    }

    // ⭐ SYNCHRONISER LES REACTIONS
    await reactionsDB.value.replicate.to('http://elia:admin@localhost:5984/reactions_elia_nicolo')

    await updateTotalLikes(postId)
    newComment.value = ''
    await fetchData()
    console.log('✅ Réaction ajoutée et synchronisée')
  } catch (err) {
    console.error('❌ Erreur ajout réaction:', err)
  }
}

const updateTotalLikes = async (postId: string) => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    const result = await reactionsDB.value.find({
      selector: {
        post_id: postId,
        isliked: true
      }
    })

    const likeCount = result.docs.length

    const post = await postsDB.value.get(postId)
    post.total_likes = likeCount
    await postsDB.value.put(post)

    // ⭐ SYNCHRONISER LE POST MIS À JOUR
    await postsDB.value.replicate.to('http://elia:admin@localhost:5984/post_elia_nicolo')

    console.log(`✅ Post ${postId} : ${likeCount} likes (synchronisé)`)
  } catch (err) {
    console.error('❌ Erreur mise à jour total_likes:', err)
  }
}

const deleteComment = async (postId: string, commentText: string) => {
  if (!reactionsDB.value) return

  const reactionId = `reaction_${postId}_user_1`
  
  try {
    const reaction = await reactionsDB.value.get(reactionId)
    reaction.comments = reaction.comments.filter(c => c !== commentText)
    await reactionsDB.value.put(reaction)
    await fetchData()
    console.log('✅ Commentaire supprimé')
  } catch (err) {
    console.error('❌ Erreur suppression commentaire:', err)
  }
}


onMounted(async() => {
  await initDatabase()
  startLiveSync()
})

const toggleOffline = (): void => {
  isOffline.value = !isOffline.value
  buttonName.value = isOffline.value ? 'Hors ligne' : 'En ligne'

  if (isOffline.value) {
    try {
      if (postsSyncHandler && typeof postsSyncHandler.cancel === 'function') postsSyncHandler.cancel()
      if (reactionsSyncHandler && typeof reactionsSyncHandler.cancel === 'function') reactionsSyncHandler.cancel()
      console.log('🔌 Sync arrêté — mode hors ligne')
    } catch (err) {
      console.error('❌ Erreur arrêt sync:', err)
    }
  } else {
    // Re-démarrer la sync live
    startLiveSync()
    console.log('🌐 Sync redémarrée — mode en ligne')
  }
}

const syncDatabase = async (): Promise<void> => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    await postsDB.value.replicate.to('http://elia:admin@localhost:5984/post_elia_nicolo')
    await reactionsDB.value.replicate.to('http://elia:admin@localhost:5984/reactions_elia_nicolo')
    showSyncMessage.value = true
    setTimeout(() => { showSyncMessage.value = false }, 2000)
    await fetchData()
    console.log('✅ Synchronisation manuelle terminée')
  } catch (err) {
    console.error('❌ Erreur sync manuelle:', err)
  }
}
</script>

<template>
  <div style="padding: 20px; max-width: 800px; margin: 0 auto;">
    <h1>Gestion Posts & Réactions</h1>

    <div style="background: #f5f5f5; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
      <button @click="syncDatabase">🔄 Synchroniser</button>
      <button @click="toggleOffline">{{ buttonName }}</button>
      <button @click="fetchData">📥 Rafraîchir</button>
      <button @click="getTopLikedPosts">⭐ Top likes</button>
      
      <div v-if="showSyncMessage" style="color: green; margin-top: 10px;">
        ✅ Synchronisation réussie
      </div>
      
      <div v-if="isOffline" style="color: red; margin-top: 10px;">
        🔴 Mode HORS LIGNE
      </div>
    </div>

    <div style="margin-bottom: 20px;">
      <input 
        v-model="searchTerm" 
        @input="searchPosts(searchTerm)"
        placeholder="🔍 Rechercher un post..." 
        style="width: 100%; padding: 10px; font-size: 16px;"
      />
    </div>

    <div style="background: #e3f2fd; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
      <h3>➕ Nouveau post</h3>
      <input 
        v-model="documentName" 
        placeholder="Nom du post" 
        style="width: 100%; margin-bottom: 10px;"
      />
      <textarea 
        v-model="documentContent" 
        placeholder="Contenu du post"
        rows="3"
        style="width: 100%; margin-bottom: 10px;"
      ></textarea>
      <button @click="addDocument">Créer le post</button>
    </div>

    <div v-if="selectedConflict" style="background: #fff3cd; padding: 20px; margin-bottom: 20px; border-radius: 8px;">
      <h2>⚠️ Résolution de conflit</h2>
      
      <div style="background: #d4edda; padding: 10px; margin: 10px 0;">
        <h4>Version locale</h4>
        <p><strong>Nom:</strong> {{ selectedConflict.post_name }}</p>
        <p><strong>Contenu:</strong> {{ selectedConflict.post_content }}</p>
        <button @click="keepLocal">Garder cette version</button>
      </div>

      <div 
        v-for="(version, idx) in otherVersions" 
        :key="idx" 
        style="background: #fff0f0; padding: 10px; margin: 10px 0;"
      >
        <h4>Version distante {{ idx + 1 }}</h4>
        <p><strong>Nom:</strong> {{ version.post_name }}</p>
        <p><strong>Contenu:</strong> {{ version.post_content }}</p>
        <button @click="keepRemote(idx)">Garder cette version</button>
      </div>

      <button @click="cancelConflictResolution" style="margin-top: 10px;">Annuler</button>
    </div>

    <div v-if="!selectedConflict">
      <h2>📄 Posts ({{ postsData.length }})</h2>

      <article 
        v-for="post in postsData" 
        :key="post._id"
        style="border: 1px solid #ccc; padding: 15px; margin: 15px 0; border-radius: 8px;"
      >
        <div 
          v-if="post._conflicts && post._conflicts.length > 0" 
          style="background: #ffebee; padding: 10px; margin-bottom: 15px; border-radius: 4px;"
        >
          <span style="color:red; font-weight: bold;">
            ⚠️ Conflit détecté ({{ post._conflicts.length }} version(s))
          </span>
          <button @click="resolveConflict(post._id)" style="margin-left: 10px;">
            Voir les versions
          </button>
        </div>

        <h3>{{ post.post_name }}</h3>
        <p>{{ post.post_content }}</p>
        <p style="font-size: 0.85em; color: #666;">
          💾 ID: {{ post._id }} | 👍 {{ post.total_likes }} likes
        </p>

        <div 
          v-if="getReactionForPost(post._id)" 
          style="background: #f9f9f9; padding: 10px; margin: 10px 0; border-radius: 4px;"
        >
          <p v-if="getReactionForPost(post._id)!.isliked" style="margin: 0 0 10px 0;">
            ✅ Vous aimez ce post
          </p>
          <p v-else style="margin: 0 0 10px 0; color: #999;">
            Vous n'avez pas liké
          </p>

          <div v-if="getReactionForPost(post._id)!.comments.length > 0">
            <p style="margin: 10px 0 5px 0;">
              <strong>💬 Commentaires ({{ getReactionForPost(post._id)!.comments.length }}) :</strong>
            </p>
            <ul style="list-style: none; padding-left: 0; margin: 0;">
              <li 
                v-for="(c, idx) in getReactionForPost(post._id)!.comments" 
                :key="idx"
                style="margin: 5px 0; padding: 8px; background: #fff; border: 1px solid #ddd; border-radius: 4px; display: flex; justify-content: space-between; align-items: center;"
              >
                <span>{{ c }}</span>
                <button 
                  @click="deleteComment(post._id, c)"
                  style="background: #f44336; color: white; padding: 4px 8px; font-size: 0.85em; border: none; border-radius: 3px; cursor: pointer;"
                >
                  🗑️
                </button>
              </li>
            </ul>
          </div>
        </div>

        <div style="background: #f0f0f0; padding: 10px; margin: 10px 0; border-radius: 4px;">
          <button @click="addReaction(post._id, undefined, true)">👍 Like</button>
          <button @click="addReaction(post._id, undefined, false)">👎 Unlike</button>
          <input 
            v-model="newComment" 
            placeholder="Ajouter un commentaire" 
            style="width: 60%;"
          />
          <button @click="addReaction(post._id, newComment)">💬 Commenter</button>
        </div>

        <div style="margin-top: 15px; padding-top: 10px; border-top: 1px solid #ddd;">
          <input 
            v-model="post.post_name"
            placeholder="Nouveau nom" 
            style="width: 60%;"
          />
          <button @click="updateDocument(post, post.post_name)">✏️ Modifier</button>
          <button 
            @click="deleteDocument(post._id, post._rev)" 
            style="background: #f44336; color: white;"
          >
            🗑️ Supprimer
          </button>
        </div>
      </article>
    </div>
  </div>
</template>

<style scoped>
button {
  margin: 5px;
  padding: 8px 12px;
  cursor: pointer;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: #fff;
  transition: background 0.2s;
}

button:hover {
  background: #e0e0e0;
}

input, textarea {
  margin: 5px;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

article {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
</style>
