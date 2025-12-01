<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

interface Post {
  _id: string
  _rev?: string
  post_name: string
  post_content: string
  total_likes?: number //counter des likes
  comments?: string[]
  attributes?: {
    creation_date: any
  }
  _conflicts?: string[]
}


interface Reaction {
  _id: string;           //  reaction_postId
  _rev?: string;         // révision
  user_id: string;      // id de l'utilisateur
  post_id: string;       // lien vers le post
  isliked: boolean;        // true si isliked
  comments: string[];    // tableau de commentaires
}


//Variables réactives
const documentName = ref('')
const documentNewName = ref<string[]>([])
const documentContent = ref('')
const storage = ref()
const postsData = ref<Post[]>([])
const showSyncMessage = ref(false)
const isOffline = ref(false)
const selectedConflict = ref<any>(null)
const otherVersions = ref<any[]>([])
const buttonName = ref("En ligne")
const searchTerm = ref('');
const newComment = ref('');
const reactionsData = ref<Reaction[]>([]);

let syncHandler: any = null

//Initialisation de la base locale et réplication initiale
const initDatabase = () => {
  console.log('=> Connexion à la base locale')
  const db = new PouchDB('collection_infradon2')
  storage.value = db

  // Réplication initiale depuis CouchDB
  db.replicate
    .from('http://elia:admin@localhost:5984/infradon2')
    .then(() => fetchData())
    .catch(err => console.error('Erreur réplication initiale:', err))

  storage.value.createIndex({
    index: { fields: ['post_name'] } //index pour la recherche par nom de post
  }).then(() => {
    console.log('Index créé sur post_name');
  });

  storage.value.createIndex({ //index pour chaque post et user
    index: { fields: ['post_id', 'user_id'] }
  }).then(() => {
    console.log('Index créé sur post_id et user_id');
  });

  storage.value.createIndex({ //index pour chaque post pour les réactions
    index: { fields: ['post_id'] }
  }).then(() => {
    console.log('Index créé sur post_id');
  });
}

// Rafraîchir (réplication ponctuelle)
const syncDatabase = () => {
  PouchDB.sync(storage.value, 'http://elia:admin@localhost:5984/infradon2')
    .then(() => {
      fetchData()
      showSyncMessage.value = true
      setTimeout(() => (showSyncMessage.value = false), 3000)
    })
    .catch(err => console.error('Erreur réplication:', err))
}

// Toggle Online/Offline
const toggleOffline = () => {
  isOffline.value = !isOffline.value
  buttonName.value = isOffline.value ? 'Hors ligne' : 'En ligne'

  if (isOffline.value) {
    // Stop live sync
    if (syncHandler) {
      syncHandler.cancel()
      console.log('Mode OFFLINE activé')
    }
  } else {
    // Restart live sync SANS résolution automatique
    syncHandler = storage.value.sync('http://elia:admin@localhost:5984/infradon2', {
      live: true,
      retry: true
    })
      .on('change', (info) => {
        console.log('Changement détecté:', info);
        fetchData(); // Rafraîchit pour voir les conflits
      })
      .on('error', (err) => {
        console.error('Erreur sync:', err);
      });

    console.log('Mode ONLINE activé')
  }
}

// Récupérer les données
const fetchData = () => {
  storage.value
    .allDocs({ include_docs: true, conflicts: true })
    .then((result: any) => {
      const allDocs = result.rows
        .map((row: any) => row.doc)
        .filter(doc => !doc._id.startsWith('_'))

      postsData.value = allDocs.filter(doc => !doc._id.startsWith('reaction_')) //données post
      reactionsData.value = allDocs.filter(doc => doc._id.startsWith('reaction_')) //données réactions

      console.log('Posts:', postsData.value.length);
      console.log('Réactions:', reactionsData.value.length);
      console.log('Documents avec conflits:', postsData.value.filter(p => p._conflicts));
      documentNewName.value = postsData.value.map(post => post.post_name)
    })
    .catch(err => console.error('Erreur récupération:', err))
};

// Résoudre conflit 
const resolveConflict = async (id: string) => {
  const doc = await storage.value.get(id, { conflicts: true });
  console.log('Version active:', doc);
  console.log('Révisions en conflit:', doc._conflicts);

  selectedConflict.value = doc;
  otherVersions.value = [];

  for (const rev of doc._conflicts) {
    const otherDoc = await storage.value.get(id, { rev });
    otherVersions.value.push(otherDoc);
  }
};

const keepLocal = async () => {
  // Supprimer toutes les versions en conflit
  for (const rev of selectedConflict.value._conflicts) {
    await storage.value.remove(selectedConflict.value._id, rev);
  }
  fetchData();
  selectedConflict.value = null;
  otherVersions.value = [];
};

const keepRemote = async (index: number) => {
  const chosenVersion = otherVersions.value[index];

  // Mettre à jour avec la version distante qui est recréer en local 
  const newDoc = {
    ...selectedConflict.value, //sturcture local
    post_name: chosenVersion.post_name,//contenu distant
    post_content: chosenVersion.post_content,
    likes: chosenVersion.likes,
    comments: chosenVersion.comments,
  };
  await storage.value.put(newDoc);
  for (const rev of selectedConflict.value._conflicts) {
    await storage.value.remove(selectedConflict.value._id, rev);
  } //supprimer les autres versions en conflit

  fetchData();
  selectedConflict.value = null;
  otherVersions.value = [];
};

// Recherche de posts par nom
const searchPosts = async (term: string) => {
  if (!term || term.trim() === '') {
    fetchData();  // Si vide, recharger tout
    return;
  }

  try {
    // l'index sur post_name
    const result = await storage.value.find({
      selector: { 
        post_name: { 
          $regex: new RegExp(term, 'i')  // 'i' = case insensitive
        } 
      }
    });
    
    postsData.value = result.docs.filter(doc => !doc._id.startsWith('reaction_'));
  } catch (err) {
    console.error('Erreur recherche:', err);
  }
};

// Ajouter un document
const addDocument = () => {
  storage.value
    .post({
      post_name: documentName.value,
      post_content: 'blabla',
      likes: 0,
      comments: []
    })
    .then(() => {
      documentName.value = '';
      fetchData();
    })
    .catch(err => console.log(err))
}

// Modifier un document
const updateDocument = (post_id: string, post_rev: string, index: number) => {
  storage.value
    .put({
      _id: post_id,
      _rev: post_rev,
      post_name: documentNewName.value[index],
      post_content: documentContent.value + ' - updated ' + new Date().toISOString()
    })
    .then(() => fetchData())
    .catch(err => console.log(err))
}

//Supprimer un document
const deleteDocument = (post_id: string, post_rev: string) => {
  storage.value
    .remove(post_id, post_rev)
    .then(() => fetchData())
    .catch(err => console.log(err))
}

// Factory pour générer 50 documents aléatoires
const generateFactory = () => {
  const docs = []
  for (let i = 0; i < 50; i++) {
    docs.push({
      _id: 'doc_' + i + '_' + Date.now(),
      post_name: 'Post ' + i,
      post_content: 'Contenu aléatoire ' + Math.random().toString(36).substring(7),
      likes: Math.floor(Math.random() * 100),
      comments: []
    })
  }
  storage.value.bulkDocs(docs).then(() => fetchData())
}

const cancelConflictResolution = () => {
  selectedConflict.value = null;
  otherVersions.value = [];
}

const addReaction = async (post_id: string, comment?: string, isliked?: boolean) => {
  try { //indexer pour chaque post et user
    const existing = await storage.value.find({
      selector: {
        post_id: post_id,
        user_id: 'user_1'
      }
    });

    if (existing.docs.length > 0) {
      // Modifier la réaction existante
      const reaction = existing.docs[0];
      if (comment && comment.trim()) {  // conserver uniquement les commentaires non vides
        reaction.comments.push(comment);
      }
      if (isliked !== undefined) {
        reaction.isliked = isliked;
      }
      await storage.value.put(reaction);
    } else { // créer une nouvelle réaction    
      const newReaction: Reaction = {
        _id: 'reaction_' + post_id + '_user1',  //ID unique
        post_id,
        user_id: 'user_1',
        isliked: isliked || false,
        comments: comment && comment.trim() ? [comment] : []
      };
      await storage.value.put(newReaction);
    }

    newComment.value = '';   // reste le champ commentaire
    fetchData();
  } catch (err) {
    console.error('Erreur ajout réaction:', err);
  }
};


const cleanReaction = async (post_id: string) => {
  const existing = await storage.value.find({
    selector: { post_id: post_id, user_id: 'user_1' }
  });
  if (existing.docs.length > 0) {
    const reaction = existing.docs[0];
    if (!reaction.isliked && reaction.comments.length === 0) {
      await storage.value.remove(reaction._id, reaction._rev);
    }
  }
};

// Version synchrone pour le template (cherche dans reactionsData déjà chargé)
const getReactionForPost = (post_id: string) => {
  return reactionsData.value.find(r => r.post_id === post_id && r.user_id === 'user_1') || null;
}
const reactionForPost = async (post_id: string) => { // Renvoie la réaction pour un post donné
  try {
    const result = await storage.value.find({
      selector: { post_id: post_id, user_id: 'user_1' }
    });
    return result.docs.length > 0 ? result.docs[0] : null; // Retourne une réaction vide si aucune n'existe
  } catch (err) {
    console.error('Erreur récupération réaction:', err);
    return null;
  }
};

  onMounted(() => {
    console.log('=> Composant initialisé')
    initDatabase()
  })
</script>

<template>
  <h1>Gestion des données</h1>

  <!-- Boutons de contrôle -->
  <div style="margin-bottom: 20px;">
    <button @click="syncDatabase">Rafraîchir les données</button>
    <button @click="toggleOffline">Vous êtes: {{ buttonName }}</button>
    <button @click="addDocument">Ajouter un document</button>
    <input v-model="documentName" placeholder="Nom du document" />
    <button @click="generateFactory">Générer 50 documents</button>
  </div>

  <p v-if="showSyncMessage" style="color: green; font-weight: bold;">
    ✓ Base synchronisée
  </p>

  <!-- Zone de résolution des conflits -->
  <div v-if="selectedConflict" style="border: 2px solid red; padding: 20px; margin: 20px 0;">
    <h3>⚠️ Résolution du conflit pour {{ selectedConflict._id }}</h3>

    <div style="background: #f0f0f0; padding: 10px; margin: 10px 0;">
      <h4>Version locale (actuelle)</h4>
      <p><strong>Nom:</strong> {{ selectedConflict.post_name }}</p>
      <p><strong>Contenu:</strong> {{ selectedConflict.post_content }}</p>
      <button @click="keepLocal()">Garder la version locale</button>
    </div>

    <div v-for="(version, idx) in otherVersions" :key="idx" style="background: #fff0f0; padding: 10px; margin: 10px 0;">
      <h4>Version distante {{ idx + 1 }}</h4>
      <p><strong>Nom:</strong> {{ version.post_name }}</p>
      <p><strong>Contenu:</strong> {{ version.post_content }}</p>
      <button @click="keepRemote(idx)">Garder cette version</button>
    </div>

    <button @click="cancelConflictResolution" style="margin-top: 10px;">Annuler</button>
  </div>

  <!-- Liste des documents -->
  <div v-if="!selectedConflict">
    <h2>Documents ({{ postsData.length }})</h2>

    <article v-for="(post, index) in postsData" :key="post._id"
      style="border: 1px solid #ccc; padding: 10px; margin: 10px 0;">

      <div>
        <button @click="addReaction(post._id, null, true)">👍 isliked</button>
        <input v-model="newComment" placeholder="Ajouter un commentaire" />
        <button @click="addReaction(post._id, newComment)">Commenter</button>
      </div>

      <p v-if="getReactionForPost(post._id)">
        Likes: {{ getReactionForPost(post._id).isliked ? 'Oui' : 'Non' }}
      </p>
      <p v-if="getReactionForPost(post._id) && getReactionForPost(post._id).comments.length">
        Premier commentaire: {{ getReactionForPost(post._id).comments[0] }}
      </p>
      <p v-if="reactionForPost(post._id).comments.length">Premier commentaire: {{ reactionForPost(post._id).comments[0]
      }}</p>
      <p v-if="reactionForPost(post._id) && reactionForPost(post._id).comments.length">
        Premier commentaire: {{ reactionForPost(post._id).comments[0] }}
      </p>


      <div v-if="post._conflicts" style="background: #ffebee; padding: 5px; margin-bottom: 10px;">
        <span style="color:red; font-weight: bold;">⚠️ Conflit détecté ({{ post._conflicts.length }} version(s))</span>
        <button @click="resolveConflict(post._id)" style="margin-left: 10px;">Voir les versions</button>
      </div>

      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
      <p style="font-size: 0.8em; color: #666;">ID: {{ post._id }}</p>

      <div style="margin-top: 10px;">
        <input v-model="documentNewName[index]" placeholder="Nouveau nom" />
        <button @click="updateDocument(post._id, post._rev, index)">Modifier</button>
        <button @click="deleteDocument(post._id, post._rev)">Supprimer</button>
      </div>
    </article>
  </div>
</template>

<style scoped>
button {
  margin: 5px;
  padding: 8px 12px;
  cursor: pointer;
}

input {
  margin: 5px;
  padding: 8px;
}

article {
  border-radius: 4px;
}
</style>