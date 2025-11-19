<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev: string
  index: number
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

const documentName = ref('')
const documentNewName = ref<string[]>([])
const documentContent = ref('')
const storage = ref()
const postsData = ref<Post[]>([])

// show or hide message when DB is synced
const showSyncMessage = ref(false)

const initDatabase = () => {
  console.log('=> Connexion à la base de donnes')
  const db = new PouchDB('collection_infradon2')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
    db.replicate
      .from('http://elia:admin@localhost:5984/infradon2')
      .then((_result) => {
        fetchData()
      })
      .catch((err) => {
        console.error('=> Erreur de réplication initiale : ', err)
        console.log('=> Réplication terminée')
      })
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

const syncDatabase = () => {
  storage.value.replicate  // Removed v-if (that's only for template!)
    .to('http://elia:admin@localhost:5984/infradon2')
    .then((_result) => {
      fetchData()
      console.log('=> Réplication terminée')

      // SHOW THE MESSAGE:
      showSyncMessage.value = true

      // HIDE IT AFTER 3 SECONDS:
      setTimeout(() => {
        showSyncMessage.value = false
      }, 3000)
    })
    .catch((err) => {
      console.error('=> Erreur de réplication initiale : ', err)
    })
}

const fetchData = () => {
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result: any) => {
      console.log(result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
      documentNewName.value = postsData.value.map((post) => post.post_name)
      postsData.value.forEach((post, index) => {
        documentNewName.value.push(post.post_name)
      })
      console.log('=> Données récupérées avec succès')
      console.log(documentNewName.value)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données: ', error)
    })
}

const addDocument = () => {
  storage.value
    .post({
      post_name: documentName.value,
      post_content: 'blabla',
    })
    .then((response) => {
      fetchData()
    })
    .catch((err) => {
      console.log(err)
    })
}

const updateDocument = (post_id, post_rev, index) => {
  storage.value
    .put({
      _id: post_id,
      _rev: post_rev,
      post_name: documentNewName.value[index],
      post_content: documentContent.value + ' - updated' + new Date().toISOString(),
    })
    .then((response) => {
      fetchData()
    })
    .catch((err) => {
      console.log(err)
    })
}

const deleteDocument = (post_id, post_rev) => {
  if (postsData.value.length === 0) {
    console.warn('No documents to delete')
    return
  }
  storage.value
    .remove(post_id, post_rev)
    .then((response: any) => {
      fetchData()
    })
    .catch((err: any) => {
      console.log(err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

</script>

<template>
  <h1>Fetch Data</h1>


  <button @click="syncDatabase">Rafraîchir les données</button>


  <p v-if="showSyncMessage" style="color: green; font-weight: bold;">
    ✓ DataBase syncronisé
  </p>

  <br>
  <input v-model="documentName" placeholder="Chosisir le nom du document" />
  <button @click="addDocument">Ajouter un document</button>
  <br />

  <ul>
    <li v-for="post in postsData" :key="(post as any).id">
      {{ post.post_name }} - {{ post.post_content }}
    </li>
  </ul>

  <article v-for="(post, index) in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>{{ post._id }}</p>
    <button @click="updateDocument(post._id, post._rev, index)">Modifier un document</button>
    <br />
    <input v-model="documentNewName[index]" placeholder="Change le nom du doc" />
    <br />
    <button @click="deleteDocument(post._id, post._rev)">Supprimer un document</button>
  </article>
</template>
