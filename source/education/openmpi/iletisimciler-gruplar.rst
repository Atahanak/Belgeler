
İletişimciler ve Gruplar
========================

Yazılım kütüphanelerinin kullanımı, programcıların her ihtiyaç duyulduğunda tekerleği yeniden icat etmek zorunda kalmadan yeni yazılım geliştirmelerine olanak tanır. Yazılım bileşenleri, düzgün bir şekilde paketlenir ve belgelenirse, programcıların amaçlarına uyacak şekilde kolayca yeniden kullanılabilir ve birleştirilebilir. MPI standardı farklı mimarilerde sorunsuz bir şekilde çalışabilmek için işlemler arasındaki iletişimi ve işlemler hakkındaki bilgileri ``communicators`` ve ``groups`` soyutlaştırmaları ile sağlar.

İletişimciler iki farklı şekilde tanımlanır:


* ``Intracommunicators``\ : Noktadan noktaya ve toplu mesaj aktarma rutinleri aracılığıyla etkileşime girebilen işlemler topluluğu.
* ``Intercommunicators``\ : Mesaj iletimi yoluyla etkileşime girebilen ayrık ``intracommunicator`` süreçlerin toplamıdır.

Bu dersin kapsamında ``intracommunicator``\ 'lar ile ilgileneceğiz ve bunlara kısacası iletişimci diye hitap edeceğiz.

Bir iletişimci temel iki objeden oluşur:


* Grup (group): sıralı bir işlem koleksiyonu. Gruptaki her işleme, negatif olmayan bir tam sayı olan bir sıra atanır. Sıralama, gruptaki her süreci benzersiz bir şekilde tanımlar.
* Bağlam (context): her bir iletişimciyi benzersiz şekilde tanımlayan sistem tanımlı bir objedir. Bağlam iletişimci için benzersiz olduğu için aynı grup birden fazla iletişimciye ait olabilir.

``MPI_COMM_WORLD`` MPI kütüphanesi tarafından önceden tanımlanmış iletişimcidir.

.. code-block:: c

   MPI_Comm comm = MPI_COMM_WORLD;

Herhangi bir iletişimcideki işlemci sayısını ve büyüklüğünü aşağıdaki fonksiyonları kullanarak elde edebiliriz.

.. code-block:: c

   int rank;
   int size;
   MPI_Comm_size(comm, &size);
   MPI_Comm_rank(comm, &rank);

MPI ile İletişimci Yaratmak
---------------------------

Yaratılmış iletişimcilerin bağlamı direk olarak değiştirilemez fakat ait olduğu grup aşağıdaki gibi çağırabilir.

.. code-block:: c

   int MPI_Comm_group(MPI_Comm comm,
                      MPI_Group *group)

Peki grupları nasıl manipüle edebiliriz?


* Hâlihazırda varolan gruplardan işlemcileri ekleyerek ve çıkararak.
* Hâlihazırda var olan gruplar üzerinde küme işlemleri, kesişim vb., kullanarak.

İşlemlerin çıkarılması ve dahil edilmesi, sıra numaraları (*ing.*, ranks) ile yapılır.
Bir işlemin bir grup içindeki sırası o işlemin benzersiz bir tanımlayıcısıdır.

.. code-block:: c

   int MPI_Group_excl(MPI_Group group,
                      int n,
                      const int ranks[],
                      MPI_Group *newgroup)

``MPI_comm_create`` fonksiyonunu yeni yarattığımız grup ile çağırdığımızda bize bu grubun arasındaki mesajlaşmayı destekleyecek bir iletişimci yaratır.

.. code-block:: c

   int MPI_Comm_create(MPI_Comm comm,
                       MPI_Group group,
                       MPI_Comm *newcomm)

Grupları manipüle etmek zahmetli ve karışık olabilir. Buna çözüm olarak bir iletişimci objesini kullanarak alt iletişimciler yaratabiliriz.

.. code-block:: c

   int MPI_Comm_split(MPI_Comm comm,
                      int color,
                      int key,
                      MPI_Comm *newcomm)

``comm``: baz olarak kullandığımız iletişimci objesi

``color``: işlemi yeni iletişimciye atama kriteri. Aynı değerleri alan işlemler yeni yaratılan iletişimcilerde aynı iletişimcide olurlar.

``key``: yeni iletişimci grubundaki arama işleminin göreli sırası.

``newcomm``: yarattığımız yeni iletişimci objesi

İletişimci Örneği
^^^^^^^^^^^^^^^^^

.. code-block:: c

   #include <stdio.h>
   #include <stdlib.h>

   #include <mpi.h>

   #define NPROCS 4

   int main(int argc, char *argv[]) {
           int rank;
           int size;
           int new_rank;
           int sendbuf;
           int recvbuf;
           int count;

           MPI_Comm new_comm;

           MPI_Init(&argc, &argv);
           MPI_Comm_rank(MPI_COMM_WORLD, &rank);
           MPI_Comm_size(MPI_COMM_WORLD, &size);
           if(rank == 0){
                   printf("MPI_COMM_WORLD size = %d\n", size);
           }

           // programda 4 işlem olup olmadığını kontrol ediyoruz
           if (size != NPROCS) {
                   fprintf(stderr, "Error: Must have %d processes in MPI_COMM_WORLD\n",
                                   NPROCS);
                   MPI_Abort(MPI_COMM_WORLD, 1);
           }

           // işlemin MPI_COMM_WORLD'dek sırasını mesaj olarak yollayacağız
           sendbuf = rank;
           count = 1;

           // işlemleri MPI_Comm_split kullnarak ikiye ayırıyoruz
           // new_comm farklı işlemler için, farklı işlem gruplarını temsil ediyor
           //işlemleri sırası ikiden büyükse farklı gruba ve küçükse farklı gruba ayırıyoruz
           int res = MPI_Comm_split(MPI_COMM_WORLD, rank < NPROCS / 2, rank, &new_comm);

           //yeni yaratılmış iletişimcinin büyüklüğüne bakıyoruz
           MPI_Comm_size(new_comm, &size);
           if(rank == 0){
                   printf("New comunicator success = %s, size = %d\n", res == MPI_SUCCESS ? "true": "false", size);
           }

           // yeni yarattığımız iletişimciyi ve orijinal iletişimciyi karşılaştırıyoruz
           int result;
           MPI_Comm_compare(MPI_COMM_WORLD,new_comm,&result);
           if(rank == 0){
                   printf("assign:    comm==copy: %d \n",result==MPI_IDENT);
                   printf("            congruent: %d \n",result==MPI_CONGRUENT);
                   printf("            not equal: %d \n",result==MPI_UNEQUAL);
           }

           // MPI_COMM_WORLD'deki sıra toplamını yeni ve küçük iletişimciyi kullanarak indirgiyoruz
           // çıktıda indirgenmiş değerler farklı gruplarda farklı olacaktır
           MPI_Allreduce(&sendbuf, &recvbuf, count, MPI_INT, MPI_SUM, new_comm);
           printf("New Comm: rank= %d newrank= %d recvbuf= %d\n", rank, new_rank, recvbuf);

           MPI_Comm_free(&new_comm);

           MPI_Finalize();

           return 0;
   }
