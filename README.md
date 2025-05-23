### 1. Modelo de Dados (Item.kt)
```kotlin
// Arquivo: app/model/Item.kt
@Entity(tableName = "itens")  // Isso faz essa classe virar uma tabela no banco de dados
data class Item(
    @PrimaryKey(autoGenerate = true) // ID que aumenta automaticamente (1, 2, 3...)
    val id: Long = 0L,               // Valor padrão 0, o banco vai gerar os números
    val nome: String,                // Nome do item (ex: "Leite")
    val quantidade: Int = 1,         // Quantidade padrão é 1
    val comprado: Boolean = false    // Começa como não comprado
)
Explicação para prova:

@Entity = Transforma a classe em tabela no banco

@PrimaryKey = Chave única para cada item

data class = Classe especial do Kotlin que guarda dados

Os valores depois do = são valores padrão

2. Banco de Dados (ItemDatabase.kt)
kotlin
// Arquivo: app/data/ItemDatabase.kt
@Database(entities = [Item::class], version = 1) // Versão 1 do banco
abstract class ItemDatabase : RoomDatabase() {
    abstract fun itemDao(): ItemDao // Acesso às operações do banco

    companion object { // Padrão Singleton (só uma instância do banco)
        @Volatile
        private var INSTANCE: ItemDatabase? = null

        fun getDatabase(context: Context): ItemDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    ItemDatabase::class.java,
                    "item_database" // Nome do arquivo do banco
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
Explicação para prova:

@Database = Define qual é o banco de dados

abstract = Não pode instanciar diretamente, o Room cria pra gente

companion object = Padrão Singleton (importante para prova!)

Room.databaseBuilder = Constrói o banco de dados

3. Operações do Banco (ItemDao.kt)
kotlin
// Arquivo: app/data/ItemDao.kt
@Dao
interface ItemDao {
    @Insert
    suspend fun insert(item: Item) // 'suspend' = função de corrotina

    @Update
    suspend fun update(item: Item)

    @Delete
    suspend fun delete(item: Item)

    @Query("SELECT * FROM itens ORDER BY nome ASC")
    fun getAllItens(): LiveData<List<Item>> // LiveData = Dados observáveis
}
Explicação para prova:

@Dao = Data Access Object (objeto que acessa o banco)

@Insert, @Update, @Delete = Operações básicas

suspend = Só roda em corrotinas (thread background)

LiveData = Atualiza automaticamente quando os dados mudam

4. ViewModel (ItemViewModel.kt)
kotlin
// Arquivo: app/ui/viewmodel/ItemViewModel.kt
class ItemViewModel(application: Application) : AndroidViewModel(application) {
    private val dao = ItemDatabase.getDatabase(application).itemDao()
    val allItens: LiveData<List<Item>> = dao.getAllItens()

    fun insert(item: Item) {
        viewModelScope.launch { // Corrotina no escopo do ViewModel
            dao.insert(item)
        }
    }

    fun update(item: Item) {
        viewModelScope.launch {
            dao.update(item)
        }
    }

    fun delete(item: Item) {
        viewModelScope.launch {
            dao.delete(item)
        }
    }
}
Explicação para prova:

AndroidViewModel = ViewModel com contexto da aplicação

viewModelScope.launch = Executa em background

LiveData = Permite a Activity observar mudanças

Faz ponte entre a UI e o banco de dados

5. Activity Principal (MainActivity.kt)
kotlin
// Arquivo: app/ui/views/MainActivity.kt
class MainActivity : AppCompatActivity() {
    private lateinit var viewModel: ItemViewModel
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Configura o ViewModel
        viewModel = ViewModelProvider(this).get(ItemViewModel::class.java)

        // Configura o RecyclerView
        val adapter = ItemAdapter { item ->
            viewModel.delete(item) // Quando clicar em um item, deleta
        }
        binding.recyclerView.adapter = adapter
        binding.recyclerView.layoutManager = LinearLayoutManager(this)

        // Observa a lista de itens
        viewModel.allItens.observe(this) { itens ->
            itens?.let { adapter.submitList(it) }
        }

        // Botão de adicionar
        binding.fab.setOnClickListener {
            val nome = binding.editText.text.toString()
            if (nome.isNotEmpty()) {
                viewModel.insert(Item(nome = nome))
                binding.editText.text.clear()
            }
        }
    }
}
Explicação para prova:

ViewModelProvider = Pega a instância do ViewModel

LiveData.observe = Atualiza a lista automaticamente

RecyclerView = Lista eficiente para muitos itens

ViewBinding = Acesso seguro aos elementos da tela

6. Adaptador (ItemAdapter.kt)
kotlin
// Arquivo: app/ui/adapter/ItemAdapter.kt
class ItemAdapter(
    private val onClick: (Item) -> Unit
) : ListAdapter<Item, ItemAdapter.ItemViewHolder>(ItemDiffCallback()) {

    class ItemViewHolder(private val binding: ItemListBinding) : 
        RecyclerView.ViewHolder(binding.root) {
        
        fun bind(item: Item) {
            binding.itemNome.text = item.nome
            binding.itemQuantidade.text = item.quantidade.toString()
            binding.root.setOnClickListener { onClick(item) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ItemViewHolder {
        val binding = ItemListBinding.inflate(
            LayoutInflater.from(parent.context),
            parent,
            false
        )
        return ItemViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ItemViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

class ItemDiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(oldItem: Item, newItem: Item): Boolean {
        return oldItem.id == newItem.id
    }

    override fun areContentsTheSame(oldItem: Item, newItem: Item): Boolean {
        return oldItem == newItem
    }
}
Explicação para prova:

ListAdapter = Adaptador inteligente para RecyclerView

DiffUtil = Compara itens para atualizar só o necessário

ViewHolder = Guarda as referências das views

onCreateViewHolder = Cria os itens da lista

onBindViewHolder = Preenche os dados em cada item
