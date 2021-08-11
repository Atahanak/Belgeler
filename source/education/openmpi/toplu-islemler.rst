
Toplu İşlemler
==============

Şu ana kadar işlemler (ya da programlar) arasındaki iletişimden bahsettik. Fakat dağıtık hesaplamada 
sadece iki işlemin birbiriyle konuşması yeterli değildir. Bazı durumlarda bütün işlemlerin bir araya gelerek 
kollektif bir operasyon ya da iletişim gerçekleştirmesi gerekebilir. MPI standardı bu özel durumların birçoğunu desteklemektedir. Bu durumlar:

* Senkronizasyon:

  * Belirtilen gruptaki bütün işlemler belirtilen bir noktaya erişene kadar birbirlerini beklerler - bütün işlemler ``MPI_COMM_WORLD`` bu operasyona ile dahil edilebilir.
  * OpenMPI standardında ``MPI_Barrier`` fonksiyonu ile uyarlanmıştır. 
  * Genellikle farklı işlemler tarafından gerçekleştirilen görevlerin istenen şekilde çizelgelemesi için kullanılır. 
  * Hata bulmak ve ayıklamak için de faydalıdır.

* Tek işlemden bütün işlemlere iletişim:

  * Bir işlem (sıra numarası ile) diğer bütün işlemlere mesaj yollar.
  * MPI standardında ``MPI_Bcast`` ve ``MPI_Scatter`` fonksiyonları ile gerçekleştirilir.

* Bütün işlemlerden tek bir işleme iletişim:

  * Bütün işlemler tek bir sıra numarasına (işleme) mesaj yollar.
  * MPI standardında ``MPI_Reduce`` ve ``MPI_Gather`` fonksiyonları ile gerçekleştirilir.

* Bütün işlemlerde bütün işlemlere iletişim:

  * Bütün işlemler hem mesaj yollarlar, hem de mesaj alırlar.
  * MPI standardında ``MPI_Alltoall`` ve ``MPI_Allgather`` fonksiyonları ile gerçekleştirilir.
  * Bütün işlemleri içeren indirgemeler önemli bir özel durumdur. Bu durum da ``MPI_Allreduce``  fonksiyonu ile uyarlanmıştır.

Elbette bu iletişim şekillerinin hepsi noktadan noktaya mesaj gönderimi ile, ``MPI_Send / MPI_Recv`` fonksiyonları ile yapılabilir. 
Ancak bu tür bir yaklaşım daha fazla adım gerektirir, daha yavaş çalışır ve MPI gerçeklemelerinde bulunan, optimize edilmiş 
toplu iletişim fonksiyonlarını kullanmaktan daha kötü bir performans verecektir.

Barrier (Bariyer)
-----------------

.. code-block:: c

   int MPI_Barrier( MPI_Comm communicator )

``communicator``: işlemler arası iletişimi sağlayan obje.

.. image:: /assets/openmpi-education/images/barrier.png
   :target: /assets/openmpi-education/images/barrier.png
   :alt: /assets/openmpi-education/images/barrier.png

Bu komut, parametre olarak verilen ``MPI_Comm`` objesinin içindeki sıra numaralarına sahip olan bütün işlemleri, hepsi
``MPI_Barrier`` komutunda senkron olana kadar bekletir. Böylece bütün işlemlerin senkronizasyonu sağlanmış olur.

Broadcast (Toplu Gönderi)
-------------------------

.. code-block:: c

   int MPI_Bcast(
       void* data,
       int count,
       MPI_Datatype datatype,
       int root,
       MPI_Comm communicator)

``MPI_Send``\ 'e benzer, ancak aynı veri sıra numarası verilmiş bir işlemden diğer grubun içerisindeki tüm işlemlere gönderilir. 
Bu fonksiyon ancak tüm işlemler ``MPI_Bcast`` fonksiyonuna ulaştığında geri döner, yani bir (``MPI_Barrier``) 
teşkil etmektedir. İletişimi başlatan, veriyi kaynağı işlemin sıra numarası ``root`` ile verilmiştir. 

.. image:: /assets/openmpi-education/images/bcast.png
   :target: /assets/openmpi-education/images/bcast.png
   :alt: /assets/openmpi-education/images/bcast.png


Scatter (Dağıtma)
-----------------

.. code-block:: c

   int MPI_Scatter(
       void* sendbuf,
       int sendcount,
       MPI_Datatype sendtype,
       void* recvbuffer,
       int recvcount,
       MPI_Datatype recvtype,
       int root,
       MPI_Comm communicator)

Sıra numarası ``root`` ile verilmiş kaynak işlemde bulunan ``sendbuf`` arabelleğindeki veriler 
parçalara bölünür ve her parça farklı bir işleme gönderilir. Yığındaki her parça, ``sendtype`` 
türünden ``sendcount`` kadar eleman içerir. Örneğin ``sendtype`` ``MPI_Int`` ve ``sendcount`` 2 ise, 
her işlem 2 tamsayı alacaktır. Alınan veriler ``recvbuf`` arabellğine yazılır. 
Bir önceki operasyonda olduğu gibi veriler sadece kaynak işlem tarafından gönderilir. 
Bu nedenle ``sendbuf``\ 'a yalnızca bu işlem tarafından ihtiyaç duyulur. Diğer işlemlerde
``sendbuf`` için boş bir işaretçi kullanılabilir.
Sonraki iki parametre, ``recvcount`` ve ``recvtype``\ , verinin alınacağı arabelleği tanımlar. 
Genellikle ``recvtype`` ``sendtype`` ile aynıdır ve ``recvcount`` ``Nranks*sendcount``\ 'tur. 
Burada ``Nranks``, ``communicator`` içerisinde bulunan işlem sayısını belirtmektedir.


.. image:: /assets/openmpi-education/images/scatter.png
   :target: /assets/openmpi-education/images/scatter.png
   :alt: /assets/openmpi-education/images/scatter.png


Gather (Toplama)
----------------

.. code-block:: c

   int MPI_Gather(
       void* sendbuf,
       int sendcount,
       MPI_Datatype sendtype,
       void* recvbuffer,
       int sendcount,
       MPI_Datatype recvtype,
       int root,
       MPI_Comm communicator)

``Gather`` operasyonu ``Scatter`` operasyonun tersi olarak düşünülebilir. 
Her işlem, ``sendbuf`` arabelleğindeki verileri ``root`` sıra numaralı hedef işleme gönderir. 
Bu işlem, sıra numaralarına göre verileri ``recvbuffer`` arebelleğinde toplar.

.. image:: /assets/openmpi-education/images/gather.png
   :target: /assets/openmpi-education/images/gather.png
   :alt: /assets/openmpi-education/images/gather.png


Reduce (İndirgeme)
------------------

.. code-block:: c

   int MPI_Reduce(
       void* sendbuf,
       void* recvbuffer,
       int count,
       MPI_Datatype datatype,
       MPI_Op op,
       int root,
       MPI_Comm communicator)


.. image:: /assets/openmpi-education/images/reduce.png
   :target: /assets/openmpi-education/images/reduce.png
   :alt: /assets/openmpi-education/images/reduce.png

``MPI_Reduce`` program akışını durdurur (``MPI_Barrier`` gibi) ve grup içerisinde toplu bir senkronizasyon oluşturur.
Bu fonksiyonun çalışmasından sonra, sıra numarası ``root`` olan hedef işlemci, ``communicator`` grubuna dahil olan bütün
işlemlerdeki değerlerin bir aritmetik ya da mantıksal operasyona göre indirgenmiş bir değerini elde eder.

MPI standardında aritmetik ve mantıksal işlemler dahil olmak üzere önceden tanımlanmış birkaç operasyon türü vardır. 
Bunlardan bazıları:

* ``MPI_SUM``\ : değerlerin toplamları.
* ``MPI_MAX``\ : maksimum değer.
* ``MPI_MIN``\ : minimum değer.
* ``MPI_PROD``\ : değerlerin çarpımları.
* ``MPI_MAXLOC``\ : maksimum değer ve bu değeri gönderen sıra.
* ``MPI_MINLOC``\ : minimum değer ve bu değeri gönderen sıra.

Yukarıda bahsedilen diğer kolektif iletişim fonksiyonları gibi ``MPI_Reduce`` da genellikle basit MPI mesaj 
gönderme fonksiyonlarını kullanarak oluşturabileceğiniz, elle yazılmış iletişimden daha hızlıdır. 
Bunun sebebi, bir MPI gerçeklemesinin çalıştığı sistemin topolojik yapısına bağlı olarak farklı algoritmalar 
uygulayabilmesidir. Bu özellikle, ``MPI_Reduce`` işlemlerinin, hesaplama yapmak için herhangi bir sırayı kullanmadan, 
yolda indirgemeler gerçekleştirmek için iletişim cihazlarını kullanabildiği, yüksek performanslı bilgi işlem için 
tasarlanmış sistemlerde geçerlidir. 

.. Bu sistemlerin nasıl inşa edildiğini Sanal Topolojiler adlı derste daha detaylı inceleyeceğiz.

Allreduce (Toplu İndirgeme)
---------------------------

.. code-block:: c

   int MPI_Allreduce(
        void* sendbuf,
        void* recvbuffer,
        int count,
        MPI_Datatype datatype,
        MPI_Op op,
        MPI_Comm communicator)

``MPI_Allreduce``, temelde ``MPI_Reduce`` ile aynı işlemleri gerçekleştirir, ancak sonuç tüm işlemlere gönderilir.

Scatter - Gather için bir Örnek
--------------------------------

.. code-block:: c

   #include "mpi.h"
   #include <stdio.h>

   int main(int argc, char **argv)
   {
       /* MPI programını başlatmak için Init fonksiyonunu çağırıyoruz */
       MPI_Init(&argc, &argv);
       MPI_Comm comm = MPI_COMM_WORLD;
       int rank, size;
       MPI_Comm_rank(comm, &rank);
       MPI_Comm_size(comm, &size);

       /* 
        işlemcilere dağıtılacak değerleri tanımlıyoruz
        bu örnekte initial_values adlı listedeki 4 değeri 4 farklı işlemciye
        dağıtıyoruz. Slurm scriptinizde 4 farklı node ve her node da 1 process 
        ayırabilirsiniz. 4ten fazla process tanımladığınız takdirde 
        program doğru çalışmayacaktır.
       */
       float initial_values[4] = { 100, -1000, 3.5, -2.25 };
       float values_to_scatter[4];
       const int rank_of_scatter_root = 0;
       if (rank == rank_of_scatter_root)
       {
           values_to_scatter[0] = initial_values[0];
           values_to_scatter[1] = initial_values[1];
           values_to_scatter[2] = initial_values[2];
           values_to_scatter[3] = initial_values[3];
       }

       /* dağıtım öncesi durumu çıktılıyoruz */
       printf("On rank %d, pre-scatter values were [%f, %f, %f, %f]\n", rank,
               values_to_scatter[0],
               values_to_scatter[1],
               values_to_scatter[2],
               values_to_scatter[3]);

       /* dağıtım işlemini gerçekleştiriyoruz */
       float scattered_value;
       MPI_Scatter(values_to_scatter, 1, MPI_FLOAT,
                   &scattered_value, 1, MPI_FLOAT,
                   rank_of_scatter_root, comm);

       /* dağıtım sonrası durumu çıktılıyoruz */
       printf("On rank %d, scattered value was %f\n", rank, scattered_value);

       /* temsili olarak dağıtılmış değer üzerinde bir işlem gerçekleştiriyoruz */
       float result = scattered_value * (rank + 1);

       /* yeni kök sıraya işlenmiş değerlerin bütün sırlardan topluyoruz */
       float gathered_values[4];
       const int rank_of_gather_root = 2;
       MPI_Gather(&result, 1, MPI_FLOAT,
                  gathered_values, 1, MPI_FLOAT,
                  rank_of_gather_root, comm);

       /* toplama işlemi sonrası durumu çıktılıyoruz */
       if (rank == rank_of_gather_root)
       {
           printf("On rank %d, gathered values were [%f, %f, %f, %f]\n", rank,
                   gathered_values[0],
                   gathered_values[1],
                   gathered_values[2],
                   gathered_values[3]);
       }

       /* yapılan işlemlerin ve kodun doğruluğunu kontrol ediyoruz */
       int success = (result == initial_values[rank] * (rank + 1));

       /* gather işleminin ve kodun doğruluğunu kontrol ediyoruz */
       if (rank == rank_of_gather_root)
       {
           success = success && ((gathered_values[0] == initial_values[0] * 1) &&
                                 (gathered_values[1] == initial_values[1] * 2) &&
                                 (gathered_values[2] == initial_values[2] * 3) &&
                                 (gathered_values[3] == initial_values[3] * 4));
       }
       if (success)
       {
           printf("SUCCESS on rank %d!\n", rank);
       }
       else
       {
           printf("Improvement needed before rank %d can report success!\n", rank);
       }

       /* MPI ortamını temizliyoruz */
       MPI_Finalize();
       return 0;
   }

Broadcast - Reduce Örnek
------------------------

.. code-block:: c

   #include "mpi.h"
   #include <stdio.h>

   int main(int argc, char **argv)
   {
       /* MPI programını başlatmak için Init fonksiyonunu çağırıyoruz */
       MPI_Init(&argc, &argv);
       MPI_Comm comm = MPI_COMM_WORLD;
       int rank, size;
       MPI_Comm_rank(comm, &rank);
       MPI_Comm_size(comm, &size);

       /* Saçılcak değerleri hazırlıyoruz */
       int expected_values[2] = { 100, -1000 };
       int values_to_broadcast[2];
       const int rank_of_root = 0;
       if (rank == rank_of_root)
       {
           values_to_broadcast[0] = expected_values[0];
           values_to_broadcast[1] = expected_values[1];
       }

       /* saçılma öncesi durumu çıktılıyoruz */
       printf("On rank %d, pre-broadcast values were [%d, %d]\n", rank,
               values_to_broadcast[0],
               values_to_broadcast[1]);

       /* saçılma işlemini gerçekleştiriyoruz */
       MPI_Bcast(values_to_broadcast, 2, MPI_INT, rank_of_root, comm);

       /* saçılma sonrası durumu çıktılıyoruz */
       printf("On rank %d, broadcast values were [%d, %d]\n", rank,
               values_to_broadcast[0],
               values_to_broadcast[1]);

       /* toplama işlemi kullanarak saçılan değerleri 
          bütün sıralar üzerinden indirgiyoruz */
       int reduced_values[2];
       MPI_Reduce(values_to_broadcast, reduced_values, 2, MPI_INT,
                  MPI_SUM, rank_of_root, comm);

       /* indirgeme sonrası durumu çıktılıyoruz */
       printf("On rank %d, reduced values were [%d, %d]\n", rank,
               reduced_values[0],
               reduced_values[1]);

       /* yapılan işlemlerin ve kodun doğruluğunu kontrol ediyoruz */
       int success = ((values_to_broadcast[0] == expected_values[0]) &&
                      (values_to_broadcast[1] == expected_values[1]));

       /* indirgeme işlemenin doğruluğunu kontrol ediyoruz */
       if (rank == rank_of_root)
       {
           success = success && ((reduced_values[0] == expected_values[0] * size) &&
                                 (reduced_values[1] == expected_values[1] * size));
       }
       if (success)
       {
           printf("SUCCESS on rank %d!\n", rank);
       }
       else
       {
           printf("Improvement needed before rank %d can report success!\n", rank);
       }

       /* MPI ortamını temizliyoruz */
       MPI_Finalize();
       return 0;
   }
