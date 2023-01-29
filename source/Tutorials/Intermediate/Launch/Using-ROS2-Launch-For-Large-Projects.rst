.. redirect-from::

    Tutorials/Launch-Files/Using-ROS2-Launch-for-Large-Projects
    Tutorials/Launch/Using-ROS2-Launch-for-Large-Projects

.. _UsingROS2LaunchForLargeProjects:

Administrar proyectos grandes
=============================

**Objetivo:** Aprender las prácticas recomendadas para administrar proyectos grandes utilizando ficheros de launch de ROS 2.

**Nivel del tutorial:** Intermedio

**Time:** 20 minutes
**Tiempo:** 20 minutos

.. contents:: Contenido
   :depth: 3
   :local:

Antecedentes
------------

Este tutorial describe algunos consejos para escribir ficheros de launch para proyectos grandes.
El enfoque esta en como estructurar ficheros de launch de tal forma de que puedan ser reutilizados tanto como sea posible en situaciones diferentes.
Adicionalmente, cubre ejemplos de uso de las diferentes herramientas de launch de ROS 2, como parámetros, ficheros YAML, remappings, namespaces, argumentos por defecto y configuraciones de RVIZ.

Prerequisitos
-------------

Este tutorial utiliza los paquetes :doc:`turtlesim <../../Beginner-CLI-Tools/Introducing-Turtlesim/Introducing-Turtlesim>` y :doc:`turtle_tf2_py <../Tf2/Introduction-To-Tf2>`.
Este tutorial también asume que tu has :doc:`creado un nuevo paquete <../../Beginner-Client-Libraries/Creating-Your-First-ROS2-Package>` de tipo de compilación ``ament_python`` llamado ``launch_tutorial``.

Introducción
------------

Aplicaciones grandes en un robot suelen implicar varios nodos interconectados, cada uno de los cuales puede tener muchos parámetros.
Las simulaciones de múltiples tortugas en el simulador de la tortuga puede servir como un buen ejemplo.
La simulación de la tortuga consiste de múltiples nodos de tortuga, la configuración del mundo, y el TF broadcaster y los nodos listener.

Entre todos los nodos, hay un gran numero de parámetros de ROS que afectan el comportamiento y la apariencia de estos nodos.
Los ficheros de launch de ROS 2 nos permiten empezar todos los nodos y configurar los parámetros correspondientes en un solo lugar.
Para el final del tutorial, tu compilarás el fichero de launch ``launch_turtlesim.launch.py`` en el paquete ``launch_tutorial``
Este fichero de launch inicializará diferentes nodos responsables de la simulación de dos turtlesim, así como de los TF broadcasters y el listener, los parámetros de carga y una configuración de RVIZ.
En este tutorial, cubriremos este fichero de launch y todas las características utilizadas.

Escribir ficheros de launch
---------------------------

1 Organización del nivel superior
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Uno de los objetivos en el proceso de escribir ficheros launch debe de ser hacerlos lo mas reutilizables posible.
Esto se puede hacer agrupando nodos y configuraciones relacionados en ficheros de launch separados.
Posteriormente, podría escribirse un fichero de launch de nivel superior dedicado específica.
Esto permitiría cambiar entre robots idénticos sin cambiar los ficheros de launch en absoluto.
Incluso un cambio como pasar de un robot real a uno simulado puede hacerse con solo unos pocos cambios.

A continuación repasaremos la estructura de un fichero de lanzamiento de nivel superior que hace esto posible.
Primeramente, crearemos un fichero de launch que llamara ficheros de launch separados.
Para hacer esto, vamos a crear un fichero ``launch_turtlesim.launch.py`` en el directorio ``/launch`` del paquete ``launch_tutorial``.

.. code-block:: Python

   import os

   from ament_index_python.packages import get_package_share_directory

   from launch import LaunchDescription
   from launch.actions import IncludeLaunchDescription
   from launch.launch_description_sources import PythonLaunchDescriptionSource


   def generate_launch_description():
      turtlesim_world_1 = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/turtlesim_world_1.launch.py'])
         )
      turtlesim_world_2 = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/turtlesim_world_2.launch.py'])
         )
      broadcaster_listener_nodes = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/broadcaster_listener.launch.py']),
         launch_arguments={'target_frame': 'carrot1'}.items(),
         )
      mimic_node = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/mimic.launch.py'])
         )
      fixed_frame_node = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/fixed_broadcaster.launch.py'])
         )
      rviz_node = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/turtlesim_rviz.launch.py'])
         )

      return LaunchDescription([
         turtlesim_world_1,
         turtlesim_world_2,
         broadcaster_listener_nodes,
         mimic_node,
         fixed_frame_node,
         rviz_node
      ])

Este fichero de launch incluye un conjunto de otros ficheros launch
Cada uno de estos ficheros de launch incluidos contiene nodos, parámetros y, posiblemente inclusiones anidadas, que pertenecen a una parte del sistema.
Para ser exactos, nosotros hacemos launch a nodos con dos mundos simulados de turtlesim, TF broadcaster, TF listener, mimic, broadcaster con frames fijos y Rviz.

.. note:: Consejo de diseño: Ficheros de launch de nivel superior deben ser cortos, consistir de inclusiones de otros ficheros correspondientes a los subcomponentes de la apliación, y comunmente parámetros cambiados.

Escribir ficheros de launch de la siguiente manera hace mas fácil el intercambio una pieza del sistema, como lo veremos mas adelante.
Sin embargo, hay casos donde algunos nodos o ficheros de launch tienen que ser empezados separadamente debido a razones de uso y rendimiento.

.. note:: Consejo de diseño: Ten en cuenta las ventanas y desventajas a la hora de decidir cuantos archivos de lanzamiento de nivel superior requiere tu aplicación.

2 Parámetros
^^^^^^^^^^^^

2.1 Configurar parámetros en un fichero de launch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Empezaremos escribiendo un fichero de launch que inicializará nuestra primer simulación de turtlesim.
Primero, crea un nuevo fichero llamado ``turtlesim_world_1.launch.py``.

.. code-block:: Python

   from launch import LaunchDescription
   from launch.actions import DeclareLaunchArgument
   from launch.substitutions import LaunchConfiguration, TextSubstitution

   from launch_ros.actions import Node


   def generate_launch_description():
      background_r_launch_arg = DeclareLaunchArgument(
         'background_r', default_value=TextSubstitution(text='0')
      )
      background_g_launch_arg = DeclareLaunchArgument(
         'background_g', default_value=TextSubstitution(text='84')
      )
      background_b_launch_arg = DeclareLaunchArgument(
         'background_b', default_value=TextSubstitution(text='122')
      )

      return LaunchDescription([
         background_r_launch_arg,
         background_g_launch_arg,
         background_b_launch_arg,
         Node(
            package='turtlesim',
            executable='turtlesim_node',
            name='sim',
            parameters=[{
               'background_r': LaunchConfiguration('background_r'),
               'background_g': LaunchConfiguration('background_g'),
               'background_b': LaunchConfiguration('background_b'),
            }]
         ),
      ])

Este fichero de launch empieza el nodo ``turtlesim_node``, el cual empieza la simulación turtlesim, con los parámetros de configuración de la simulación que son definidos y pasados a los nodos.

2.2 Cargar parámetros de un fichero YAML
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

En el segundo launch, empezaremos una segunda simulación de turtlesim con diferentes configuraciones.
Ahora crea un fichero ``turtlesim_world_2.launch.py``.

.. code-block:: Python

   import os

   from ament_index_python.packages import get_package_share_directory

   from launch import LaunchDescription
   from launch_ros.actions import Node


   def generate_launch_description():
      config = os.path.join(
         get_package_share_directory('launch_tutorial'),
         'config',
         'turtlesim.yaml'
         )

      return LaunchDescription([
         Node(
            package='turtlesim',
            executable='turtlesim_node',
            namespace='turtlesim2',
            name='sim',
            parameters=[config]
         )
      ])

Este fichero launch empezará el mismo nodo ``turtlesim_node`` con parámetros cuyos valores son cargados directamente desde un fichero de configuración YAML.
Definir argumentos y parámetros en ficheros YAML hace mas fácil el almacenar y cargar un numero grande de variables.
Además, ficheros YAML pueden ser exportados fácilmente desde la lista actual de ``ros2 param``.
Para aprender como hace esto, consulta el tutorial :doc:`Entender parámetros <../../Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters>`.

Vamos a ahora a crear un fichero de configuración ``turtlesim.yaml``, en el directorio ``/config`` de nuestro paquete, el cual será cargado por nuestro fichero de launch.

.. code-block:: YAML

   /turtlesim2/sim:
      ros__parameters:
         background_b: 255
         background_g: 86
         background_r: 150

Si ahora nosotros empezamos el fichero de launch ``turtlesim_world_2.launch.py``, empezaremos el ``turtlesim_node`` con los colores de fondo pre-configurados.

Para aprender mas acerca de como usar parametros y ficheros YAML, revisa el tutorual :doc:`Entender parámetros <../../Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters>`.

2.3 Usar wildcards en ficheros YAML
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Hay casos cuando nosotros queremos configurar los mismos parametros en mas de un nodo.
Estos nodos pueden tener diferentes namespaces o nombres, pero aun así tener los mismos parámetros.
Definir ficheros YAML separados para explícitamente definir namespaces y nombres para los nodos no es eficiente.
Una solución es usar caracteres wildcard, los cuales actual como substituciones para caracteres desconocidos en un valor de texto, para aplicar parámetros a diferentes nodos.

Ahora vamos a crear un nuevo fichero ``turtlesim_world_3.launch.py`` similar a ``turtlesim_world_2.launch.py`` para incluir un nodo mas de ``turtlesim_node``.

.. code-block:: Python

   ...
   Node(
      package='turtlesim',
      executable='turtlesim_node',
      namespace='turtlesim3',
      name='sim',
      parameters=[config]
   )

Cargar el mismo fichero YAML, sin embargo, no afectara la aparienca del tercer mundo de turtlesim.
La razón es que sus parámetros están almacenos dentro de otro namespace, como se muestra abajo:

.. code-block:: console

   /turtlesim3/sim:
      background_b
      background_g
      background_r

Por lo tanto, en lugar de crear una nueva configuración para el mismo nodo que usa los mismos parámetros, nosotros usaremos la sintaxis de wildcards.
``/**`` asignará todos los parametros en cada nodo, a pesar de los diferentes nombres y namespaces.

Ahora actualizaremos el ``turtlesim.yaml``, en el directorio ``/config`` de la siguiente manera:

.. code-block:: YAML

   /**:
      ros__parameters:
         background_b: 255
         background_g: 86
         background_r: 150

Ahora incluye la descripción de launch de ``turtlesim_world_3.launch.py`` en nuestro fichero de launch principal.
Usar ese fichero de configuración en nuestras descripciones de launch asignará los parámetros ``background_b``, ``background_g`` y ``background_r`` a los valores especificados en los nodos ``turtlesim3/sim`` y ``turtlesim2/sim``.

3 Namespaces
^^^^^^^^^^^^

As you may have noticed, we have defined the namespace for the turlesim world in the ``turtlesim_world_2.launch.py`` file.
Unique namespaces allow the system to start two similar nodes without node name or topic name conflicts.

.. code-block:: Python

   namespace='turtlesim2',

However, if the launch file contains a large number of nodes, defining namespaces for each of them can become tedious.
To solve that issue, the ``PushROSNamespace`` action can be used to define the global namespace for each launch file description.
Every nested node will inherit that namespace automatically.

To do that, firstly, we need to remove the ``namespace='turtlesim2'`` line from the ``turtlesim_world_2.launch.py`` file.
Afterwards, we need to update the ``launch_turtlesim.launch.py`` to include the following lines:

.. code-block:: Python

   from launch.actions import GroupAction
   from launch_ros.actions import PushROSNamespace

      ...
      turtlesim_world_2 = IncludeLaunchDescription(
         PythonLaunchDescriptionSource([os.path.join(
            get_package_share_directory('launch_tutorial'), 'launch'),
            '/turtlesim_world_2.launch.py'])
         )
      turtlesim_world_2_with_namespace = GroupAction(
        actions=[
            PushROSNamespace('turtlesim2'),
            turtlesim_world_2,
         ]
      )

Finally, we replace the ``turtlesim_world_2`` to ``turtlesim_world_2_with_namespace`` in the ``return LaunchDescription`` statement.
As a result, each node in the ``turtlesim_world_2.launch.py`` launch description will have a ``turtlesim2`` namespace.

4 Reusing nodes
^^^^^^^^^^^^^^^

Now create a ``broadcaster_listener.launch.py`` file.

.. code-block:: Python

   from launch import LaunchDescription
   from launch.actions import DeclareLaunchArgument
   from launch.substitutions import LaunchConfiguration

   from launch_ros.actions import Node


   def generate_launch_description():
      return LaunchDescription([
         DeclareLaunchArgument(
            'target_frame', default_value='turtle1',
            description='Target frame name.'
         ),
         Node(
            package='turtle_tf2_py',
            executable='turtle_tf2_broadcaster',
            name='broadcaster1',
            parameters=[
               {'turtlename': 'turtle1'}
            ]
         ),
         Node(
            package='turtle_tf2_py',
            executable='turtle_tf2_broadcaster',
            name='broadcaster2',
            parameters=[
               {'turtlename': 'turtle2'}
            ]
         ),
         Node(
            package='turtle_tf2_py',
            executable='turtle_tf2_listener',
            name='listener',
            parameters=[
               {'target_frame': LaunchConfiguration('target_frame')}
            ]
         ),
      ])


In this file, we have declared the ``target_frame`` launch argument with a default value of ``turtle1``.
The default value means that the launch file can receive an argument to forward to its nodes, or in case the argument is not provided, it will pass the default value to its nodes.

Afterwards, we use the ``turtle_tf2_broadcaster`` node two times using different names and parameters during launch.
This allows us to duplicate the same node without conflicts.

We also start a ``turtle_tf2_listener`` node and set its ``target_frame`` parameter that we declared and acquired above.

5 Parameter overrides
^^^^^^^^^^^^^^^^^^^^^

Recall that we called the ``broadcaster_listener.launch.py`` file in our top-level launch file.
In addition to that, we have passed it ``target_frame`` launch argument as shown below:

.. code-block:: Python

   broadcaster_listener_nodes = IncludeLaunchDescription(
      PythonLaunchDescriptionSource([os.path.join(
         get_package_share_directory('launch_tutorial'), 'launch'),
         '/broadcaster_listener.launch.py']),
      launch_arguments={'target_frame': 'carrot1'}.items(),
      )

This syntax allows us to change the default goal target frame to ``carrot1``.
If you would like ``turtle2`` to follow ``turtle1`` instead of the ``carrot1``, just remove the line that defines ``launch_arguments``.
This will assign ``target_frame`` its default value, which is ``turtle1``.

6 Remapping
^^^^^^^^^^^

Now create a ``mimic.launch.py`` file.

.. code-block:: Python

   from launch import LaunchDescription
   from launch_ros.actions import Node


   def generate_launch_description():
      return LaunchDescription([
         Node(
            package='turtlesim',
            executable='mimic',
            name='mimic',
            remappings=[
               ('/input/pose', '/turtle2/pose'),
               ('/output/cmd_vel', '/turtlesim2/turtle1/cmd_vel'),
            ]
         )
      ])

This launch file will start the ``mimic`` node, which will give commands to one turtlesim to follow the other.
The node is designed to receive the target pose on the topic ``/input/pose``.
In our case, we want to remap the target pose from ``/turtle2/pose`` topic.
Finally, we remap the ``/output/cmd_vel`` topic to ``/turtlesim2/turtle1/cmd_vel``.
This way ``turtle1`` in our ``turtlesim2`` simulation world will follow ``turtle2`` in our initial turtlesim world.

7 Config files
^^^^^^^^^^^^^^

Let's now create a file called ``turtlesim_rviz.launch.py``.

.. code-block:: Python

   import os

   from ament_index_python.packages import get_package_share_directory

   from launch import LaunchDescription
   from launch_ros.actions import Node


   def generate_launch_description():
      rviz_config = os.path.join(
         get_package_share_directory('turtle_tf2_py'),
         'rviz',
         'turtle_rviz.rviz'
         )

      return LaunchDescription([
         Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            arguments=['-d', rviz_config]
         )
      ])

This launch file will start the RViz with the configuration file defined in the ``turtle_tf2_py`` package.
This RViz configuration will set the world frame, enable TF visualization, and start RViz with a top-down view.

8 Environment Variables
^^^^^^^^^^^^^^^^^^^^^^^

Let's now create the last launch file called ``fixed_broadcaster.launch.py`` in our package.

.. code-block:: Python

   from launch import LaunchDescription
   from launch.actions import DeclareLaunchArgument
   from launch.substitutions import EnvironmentVariable, LaunchConfiguration
   from launch_ros.actions import Node


   def generate_launch_description():
      return LaunchDescription([
         DeclareLaunchArgument(
               'node_prefix',
               default_value=[EnvironmentVariable('USER'), '_'],
               description='prefix for node name'
         ),
         Node(
               package='turtle_tf2_py',
               executable='fixed_frame_tf2_broadcaster',
               name=[LaunchConfiguration('node_prefix'), 'fixed_broadcaster'],
         ),
      ])

This launch file shows the way environment variables can be called inside the launch files.
Environment variables can be used to define or push namespaces for distinguishing nodes on different computers or robots.

Running launch files
--------------------

1 Update setup.py
^^^^^^^^^^^^^^^^^

Open ``setup.py`` and add the following lines so that the launch files from the ``launch/`` folder and configuration file from the ``config/`` would be installed.
The ``data_files`` field should now look like this:

.. code-block:: Python

   data_files=[
         ...
         (os.path.join('share', package_name, 'launch'),
            glob(os.path.join('launch', '*.launch.py'))),
         (os.path.join('share', package_name, 'config'),
            glob(os.path.join('config', '*.yaml'))),
      ],

2 Build and run
^^^^^^^^^^^^^^^

To finally see the result of our code, build the package and launch the top-level launch file using the following command:

.. code-block:: console

   ros2 launch launch_tutorial launch_turtlesim.launch.py

You will now see the two turtlesim simulations started.
There are two turtles in the first one and one in the second one.
In the first simulation, ``turtle2`` is spawned in the bottom-left part of the world.
Its aim is to reach the ``carrot1`` frame which is five meters away on the x-axis relative to the ``turtle1`` frame.

The ``turtlesim2/turtle1`` in the second is designed to mimic the behavior of the ``turtle2``.

If you want to control the ``turtle1``, run the teleop node.

.. code-block:: console

   ros2 run turtlesim turtle_teleop_key

As a result, you will see a similar picture:

.. image:: images/turtlesim_worlds.png

In addition to that, the RViz should have started.
It will show all turtle frames relative to the ``world`` frame, whose origin is at the bottom-left corner.

.. image:: images/turtlesim_rviz.png

Summary
-------

In this tutorial, you learned about various tips and practices of managing large projects using ROS 2 launch files.
