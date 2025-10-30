<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
declare interface Post {
  _id: string
  _rev: string
  index: number
  post_name: string
  post_content: string // on verra que la si un document n'as pas de content par ex ça ne crée aucun problème
  attributes: {
    creation_date: any
  }
}
const documentName = ref('')
const documentNewName = ref<string[]>([])
// Référence à la base de données
const documentContent = ref('')
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])
const initDatabase = () => {
  console.log('=> Connexion à la base de donnes')
  const db = new PouchDB('http://elia:admin@localhost:5984/infradon2')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}
//Récupérer les données
// https://pouchdb.com/api.html#batch_fetch
const fetchData = () => {
  storage.value // changer db par storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result: any) => {
      console.log(result.rows) // Extract documents from the rows and store them in postsData
      postsData.value = result.rows.map((row: any) => row.doc)
      documentNewName.value = postsData.value.map((post) => post.post_name)
      postsData.value.forEach((post, index) => {
        documentNewName.value.push(post.post_name) // Initialize documentNewName for each post
      })
      console.log('=> Données récupérées avec succès')
      console.log(documentNewName.value)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données: ', error)
    })
}
const addDocument = () => {
  storage.value // changer db par storage.value ça appelle la globale correspondant à notre db
    .post({
      post_name: documentName.value,
      post_content: 'blabla',
    })
    .then((response) => {
      fetchData() // Refresh the data after adding
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
      fetchData() // Refresh the data after updating
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
      fetchData() // Refresh the data after deleting
    })
    .catch((err: any) => {
      console.log(err)
    })
}
onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>
<template>
   
  <h1>Fetch Data</h1>
    <input v-model="documentName" placeholder="Chosisir le nom du document" />  
  <button @click="addDocument">Ajouter un document</button>   <br />
   
  <!-- pour que les boutons s'affichent pour chaque document-->
   
  <ul>
       
    <li v-for="post in postsData" :key="(post as any).id">
            {{ post.post_name }} - {{ post.post_content }}    
    </li>
     
  </ul>
   
  <!--itérateur index chaque post avec un index// itérateur index chaque post avec un index-->
   
  <article v-for="(post, index) in postsData" v-bind:key="(post as any).id">
       
    <h2>{{ post.post_name }}</h2>
       
    <p>{{ post.post_content }}</p>
       
    <p>{{ post._id }}</p>
        <button @click="updateDocument(post._id, post._rev, index)">Modifier un document</button>  
      <br />
        <input v-model="documentNewName[index]" placeholder="Change le nom du doc" />     <br />
        <button @click="deleteDocument(post._id, post._rev)">Supprimer un document</button>  
  </article>
</template>
