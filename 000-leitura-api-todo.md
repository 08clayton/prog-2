*APi.ts*
```typescript
import todo from "./core.ts"; //Puxa tudo do arquivo core.ts, é ali que ficam as funções de gerenciar a lista.

//Cria e sobe um servidor HTTP usando o Bun. 
const server = Bun.serve({
  port: 3000,

  //Aqui ficam todas as "URLs" que o servidor conhece e sabe responder.
  routes: {
    "/": new Response(Bun.file("./public/index.html")), //Quando alguém acessa a raiz (/), manda o arquivo HTML.

    // Rotas do todo
    "/api/todo": {
      GET: async () => {
        const items = await todo.getItems()  // pede os itens pro core.ts
        return Response.json(items)   // devolve como JSON pro cliente
      },
     // adiciona um item novo na lista
      POST: async (req) => {
        const data = await req.json() as any;  // lê o corpo da requisição como JSON
        const item = data.item || null;  // pega o campo "item", ou null se não veio

       // Se não mandou o item, devolve erro 400
        if (!item)
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 });
        await todo.addItem(item);  // adiciona o item na lista
        return Response.json(data);   // devolve o que foi recebido como confirmação
      },
    },

      //Rotas com parâmetro, index é o número do item na lista
    "/api/todo/:index": {
      //atualiza um item pelo índice
      PUT: async (req) => {
        const index = parseInt(req.params.index);  // pega o index da URL e converte pra número
        // Se não for um número, manda erro
        if (isNaN(index))
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 });
        const data = await req.json() as any;  // lê o corpo da requisição
        const newItem = data.newItem || null;  // pega o campo "newItem"

        // Se não mandou o novo texto, manda erro
        if (!newItem)
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 });
        try {
          await todo.updateItem(index, newItem);  // tenta atualizar o item no índice
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });  // Se o core.ts jogou um erro, devolve pro cliente
        }
      },

      //remove um item pelo índice
      DELETE: async (req) => {
        const index = parseInt(req.params.index);// converte o index pra número
        if (isNaN(index))
          return Response.json('Índice inválido.', { status: 400 });
        try {
          await todo.removeItem(index);  //// tenta remover o item
          return Response.json(`Item no índice ${index} removido com sucesso.`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });
        }
      },
    },

    // EXEMPLO BÁSICO

    "/api/exemplo": {
      //GET simples
      GET: () => {
        return new Response(`Esse é o exemplo: ${Date.now()}`)
      },

      // POST — pega o que veio no body, gruda a data de hoje e devolve tudo
      POST: async (req) => {
        const data = await req.json() as any;
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");
        return Response.json(data);
      },
    },

    "/api/exemplo/:id": {

   // PUT — pega o id da URL, gruda no body junto com a data e devolve
      PUT: async (req, params) => {
        const { id } = req.params;
        const data = await req.json() as any;
        data.id = id;
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");
        return Response.json(data);
      },

      // PATCH — igual ao PUT, mas também lista quais campos foram mandados
      PATCH: async (req, params) => {
        const { id } = req.params;
        const data = await req.json() as any;
        data.chavesAtualizadas = Object.keys(data);
        data.id = id;
        data.atualizadoEm = new Date().toLocaleDateString("pt-BR");
        return Response.json(data);
      },

      // DELETE — só devolve uma mensagem dizendo que deletou (sem mexer em nada de verdade)
      DELETE: (req, params) => {
        const { id } = req.params;
        return new Response(`Recurso com id ${id} deletado`, { status: 200 });
      }
    }
    // FIM DO EXEMPLO BÁSICO
  },

   // Qualquer rota que não foi definida acima cai aqui — devolve 404
  async fetch(req) {
    return new Response(`Not Found`, { status: 404 });
  },
});

// Printa no terminal que o servidor tá rodando e em qual porta
console.log(`Server running at http://localhost:${server.port}`);
```
---
*core.ts*
```typescript


const jsonFilePath = __dirname + '/data.temp.json';  // Caminho do arquivo onde os dados ficam salvos
const list: string[] = await loadFromFile();  // Carrega a lista do arquivo já na inicialização (top-level await — funciona no Bun/ESM)

// Tenta ler o arquivo JSON e retornar como array de strings
async function loadFromFile() {
  try {
    const file = Bun.file(jsonFilePath);  // aponta pro arquivo (ainda não leu)
    const content = await file.text();  // lê o conteúdo como texto
    return JSON.parse(content) as string[];  // converte de JSON pra array JS
  } catch (error: any) {
    if (error.code === 'ENOENT')  // ENOENT = arquivo não existe ainda  
      return [];  // começa com lista vazia
    throw error;  // qualquer outro erro
  }
}

// Salva o estado atual da lista no arquivo JSON
async function saveToFile() {
  try {
    await Bun.write(jsonFilePath, JSON.stringify(list));  // serializa e escreve no disco
  } catch (error: any) {
   throw new Error("Erro ao salvar os dados no arquivo: " + error.message);
  }
}

// Adiciona um item no final da lista e salva
async function addItem(item: string) {
  list.push(item);  // empurra no array em memória
  await saveToFile();  // persiste no disco
}

// Simplesmente devolve a lista que já tá em memória
async function getItems() {
  return list;
}

// Substitui o item no índice informado
async function updateItem(index: number, newItem: string) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");  // valida antes de tentar acessar
  list[index] = newItem;  // troca o valor
  await saveToFile();
}

// Remove um item da lista pelo índice
async function removeItem(index: number) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");
  list.splice(index, 1);  // splice remove 1 elemento a partir do índice
  await saveToFile();
}
 
// Exporta as funções pra quem importar esse arquivo (o servidor lá em cima)
export default { addItem, getItems, updateItem, removeItem };
```
