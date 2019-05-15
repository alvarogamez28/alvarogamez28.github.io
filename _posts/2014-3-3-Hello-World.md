---
layout: post
title: Inyeccion de dependencias
---

Nuestra hoja de ruta consiste en ver:

¿Qué es una dependencia?
¿Qué es inyectar una dependencia?
Ventajas de la inyección de dependencias
Problema de la construcción de dependencias: Unity como contenedor de ID
Tutorial I: Unity en la capa de la Web Api
Tutorial II: Unity para el resto de nuestro sistema
¿Qué es una dependencia?
En software, cuando hablamos de que dos piezas, componentes, librerías, módulos, clases, funciones (o lo que se nos pueda ocurrir relacionado al área), son dependientes entre sí, nos estamos refiriendo a que uno requiere del otro para funcionar. A nivel de clases, significa que una cierta 'Clase A' tiene algún tipo de relación con una 'Clase B', delegándole el flujo de ejecución a la misma en cierta lógica.

De forma más concreta, una clase que se encarga de resolver la lógica de negocio de las mascotas en LUPI, llamémosla PetsBusinessLogic, debe interactuar con alguna clase que le permita mediar con la base de datos, llamémosla PetsRepository. Cuando queramos agregar una nueva mascota, la lógica de negocio PetsBusinessLogic deberá delegar el flujo a PetsRepository, para agregarla en la BD.

En consecuencia, PetsBusinessLogic depende de PetsRepository, o PetsBusinessLogic ---> PetsRepository.

A nivel de código, esta puede ser una fórma válida de establecer una dependencia:

  public class BreedsBusinessLogic
  {
        public BreedsRepository breedsRepository;

        public BreedsBusinessLogic()
        {
            breedsRepository = new BreedsRepository();
        }
   }
¿Qué es inyectar una dependencia?
Actualmente nuestro diseño para Lupi consiste en diferentes proyectos, cada uno representando una .dll diferente, donde las dependencias son algo así:

Lupi.Web.Api ----> Lupi.BusinessLogic ----> Lupi.Repository ---> Lupi.DataAccess
Si bien la estructura es correcta, ¿cómo están las responsabilidades a nivel de cada módulo? En un principio parece bien, pero analicemos nuevamente el la forma en que las capas se comunican:

Web Api -> Business Logic
 public class BreedsController : ApiController
 {
        private IBreedsBusinessLogic breedsBusinessLogic { get; set; }

        public BreedsController()
        {
            breedsBusinessLogic = new BreedsBusinessLogic();
        }
 }
Business Logic -> Repository
 public class BreedsBusinessLogic : IBreedsBusinessLogic
 {
       //Posible mejora de esta clase:
       //Manejar un unico contexto para unificar las transacciones realizadas sobre Breeds a partir de una Unit Of Work

       public IBreedsRepository breedsRepository;

       public BreedsBusinessLogic()
       {
           breedsRepository = new BreedsRepository();
       }
 }
¿Notaron el problema (común entre ambas porciones de código) que existe?

El problema reside en que ambas piezas de código tiene la responsabilidad de la instanciación de sus dependencias. Nuestras capas no deberían estar tan fuertemente acopladas y no deberíam ser tan dependientes entre sí. Si bien el acoplamiento es a nivel de interfaz (tenemos IBreedsBusinessLogic y IBreedsRepository), la tarea de creación/instanciación/"hacer el new" de los objetos debería ser asignada a alguien más. Nuestras capas no deberían preocuparse sobre la creación de sus dependencias.

¿Por qué? ¿Qué tiene esto de malo?👎

Si queremos reemplazar por ejemplo nuestro BreedsBusinessLogic por una implementación diferente, deberamos modificar nuestro controller. Si queremos reemplazar nuestro BreedsRepository por otro, tenemos que modificar nuestra clase BreedsBusinessLogic.

Si la BreedsBusinessLogic tiene sus propias dependencias, debemos configurarlas dentro del controller. Para un proyecto grande con muchos controllers, el código de configuración empieza a esparcirse a lo largo de toda la solución.

Es muy difícil de testear, ya que las dependencias 'estan hardcodeadas'. Nuestro controller siempre llama a la misma lógica de negocio, y nuestra lógica de negocio siempre llama al mismo repositorio para interactuar con la base de datos. En una prueba unitaria, se necesitaría realizar un mock/stub las clases dependientes, para evitar probar las dependencias. Por ejemplo: si queremos probar la lógica de BreedsBusinessLogic sin tener que depender de la lógica de la base de datos, podemos hacer un mock de BreedsRepository. Sin embargo, con nuestro diseño actual, al estar las dependencias 'hardcodeadas', esto no es posible.

Una forma de resolver esto es a partir de lo que se llama, Inyeccion de Dependencias. Vamos a inyectar la dependencia de la lógica de negocio en nuestro controller, y vamos a inyectar la dependencia del repositorio de datos en nuestra lógica de negocio. Inyectar dependencias es entonces pasarle la referencia de un objeto a un cliente, al objeto dependiente (el que tiene la dependencia). Significa simplemente que la dependencia es encajada/empujada en la clase desde afuera. Esto significa que no debemos instanciar (hacer new), dependencias, dentro de la clase.

Esto lo haremos a partir de un parámetro en el constructor, o de un setter. Por ejemplo:

 public class BreedsBusinessLogic : IBreedsBusinessLogic
 {
       public IBreedsRepository breedsRepository;

       public BreedsBusinessLogic(IBreedsRepository breedsRepository)
       {
           this.breedsRepository = breedsRepository;
       }
 }
Esto es fácil lograrlo usando interfaces o clases abstractas en C#.Siempre que una clase satisfaga la interfaz,voy a poder sustituirla e inyectarla.

Ventajas de ID 💣
Logramos resolver lo que antes habíamos descrito como desventajas o problemas 😁 👍.

Código más limpio. El código es más fácil de leer y de usar.
Nuestro software termina siendo más fácil de Testear.
Es más fácil de modificar. Nuestros módulos son flexibles a usar otras implementaciones. Desacoplamos nuestras capas.
Permite NO Violar SRP. Permite que sea más fácil romper la funcionalidad coherente en cada interfaz. Ahora nuestra lógica de creación de objetos no va a estar relacionada a la lógica de cada módulo. Cada módulo solo usa sus dependencias, no se encarga de inicializarlas ni conocer cada una de forma particular.
Permite NO Violar OCP. Por todo lo anterior, nuestro código es abierto a la extensión y cerrado a la modificación. El acoplamiento entre módulos o clases es siempre a nivel de interfaz.
Problema de la construcción de dependencias: Unity como contenedor de ID
Vimos como inyectar dependencias a través del constructor. Sin embargo, ahora tenemos un problema, el cuál es dónde construir nuestras dependencias (dónde hacer el new).

A nuestro BreedsController, no le podemos pasar por parámetro la referencia a IBreedsBusinessLogic, ya que nunca llamamos al constructor del controller explícitamente. Esto lo hace Web API, en el momento en el que se rutea la request. Y nuestra WebAPI no sabe cómo se le pasa el IBreedsBusinessLogic. Es aquí entonces donde interviene el Web API Dependency Resolver (IDependencyResolver). Veremos esto en unos momentos.

Capas inferiores, como la de BusinessLogic, son llamadas por capas superiores. Esto significa que el parámetro en la construcción tiene que venir por una capa superior. Y en ese caso, la capa superior se debe encargar de la instanciación, lo cual no es bueno. Por ejemplo: como BreedsController utiliza un IBreedsBusinessLogic, este debe crearlo. Sin embargo, a la hora de crearlo debe pasarle un repositorio, ya que la lógica de neogico precisa de el repositorio de acceso a datos para funcionar. Esto no es bueno ya que implicaría que el controller de la Web Api tenga que instanciar un repositorio, es decir, que la Web Api tenga una referencia a la forma de acceder a la base de datos.

¿Cómo resolvemos entonces este problema? 😭

Como mencionamos anteriormente, Web API define la interfaz IDependencyResolver para resolver dependencias.

public interface IDependencyResolver : IDependencyScope, IDisposable
{
    IDependencyScope BeginScope();
}

public interface IDependencyScope : IDisposable
{
    object GetService(Type serviceType);
    IEnumerable<object> GetServices(Type serviceType);
}
Cuando Web API crea un Controller, llama IDependecyResolver.GetService, pasando el tipo de controller. Esto nos da la posibilidad de implementar la interfaz para crear el controller, resolviendo las dependencias. Si el GetService retorna null, Web API busca el constructor sin parámetros de la clase Controller.

Pero en lugar de tener que hacer nuestra propia implementación IDependencyResolver, lo que haremos será usar un Contenedor de Inyección de dependencias. Estos contenedores son componentes que se encargan de administrar nuestras dependencias. Simplemente funcionan como un diccionario clave-valor; donde para cada Interfaz, existe asociada una Implementación que la resuelve. Nosotros nos vamos a encargar de registrar los tipos en el contenedor, y luego usar tal contenedor para crear objetos. Muchos de estos contenedores son tan pro 😎 que te permiten manejar el scope y el tiempo que queremos que estos vivan.

El concepto de contenedores se basa en un patrón que se llama Inversion of Control, que consiste en que un framework particular sea el que llame a nuestra aplicación, y no nosotros manualmente. Un contenedor de IoC, como el que vamos a usar, construye los objetos por nosotros, invirtiendo el flujo "usual" de control. De ahí su nombre 😜.

Ver más sobre el patrón aquí https://msdn.microsoft.com/en-us/library/ff921087.aspx?f=255&MSPPError=-2147217396

El framework que vamos a usar como contenedor se llama Unity Application Block.

Tutorial I: Unity en la capa de la Web API
1. Instalar Unity
Instalamos Unity como paquete de Nuget sobre el proyecto Lupi.Web.Api

También podemos hacerlo a través de la Consola de Administrador de Paquetes:

Install-Package Unity
Vemos que se agregarán tres referencias:

Microsoft.Practices.Unity
Microsoft.Practices.Unity.Configuration
Microsoft.Practices.Unity.RegistrationByConvention
2. Creamos nuestra implementación de IDependencyResolver
En nuestro proyecto de Lupi.Web.Api, creamos una clase UnityResolver, este será llamado por nuestra Web Api para resolver sus dependencias. Como vimos antes, implementará el contrato definido por IDependencyResolver. Definámosla de la sigueinte manera:

using Microsoft.Practices.Unity;
using System;
using System.Collections.Generic;
using System.Web.Http.Dependencies;

public class UnityResolver : IDependencyResolver
{
    protected IUnityContainer container;

    public UnityResolver(IUnityContainer container)
    {
        if (container == null)
        {
            throw new ArgumentNullException("container");
        }
        this.container = container;
    }

    public object GetService(Type serviceType)
    {
        try
        {
            return container.Resolve(serviceType);
        }
        catch (ResolutionFailedException)
        {
            return null;
        }
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        try
        {
            return container.ResolveAll(serviceType);
        }
        catch (ResolutionFailedException)
        {
            return new List<object>();
        }
    }

    public IDependencyScope BeginScope()
    {
        var child = container.CreateChildContainer();
        return new UnityResolver(child);
    }

    public void Dispose()
    {
        container.Dispose();
    }
}
3. Configurando el Dependency Resolver para que sea usado por nuestra Web Api
Lo que haremos es asignar el DependencyResolver nuestro en la Property 'DependencyResolver' del objeto HttpConfiguration global. Esto lo hacemos dentro del método Register en WebApiConfig.cs. Y lo que hacemos es agregar estas lí neas:

var container = new UnityContainer();
container.RegisterType<IBreedsBusinessLogic, BreedsBusinessLogic>(new HierarchicalLifetimeManager());
//Acá registraramos más tipos (interfaz-implementación)
//container.RegisterType<IPetsBusinessLogic, PetsBusinessLogic>(new HierarchicalLifetimeManager());
config.DependencyResolver = new UnityResolver(container);
Nuestra clase WebApiConfig.cs quedara algo así:

public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
        // Configuración y servicios de API web
        var container = new UnityContainer();
        container.RegisterType<IBreedsBusinessLogic, BreedsBusinessLogic>(new HierarchicalLifetimeManager());
        //Acá registraramos más tipos (interfaz-implementación)
        //container.RegisterType<IPetsBusinessLogic, PetsBusinessLogic>(new HierarchicalLifetimeManager());
        config.DependencyResolver = new UnityResolver(container);

        // Rutas de API web
        config.MapHttpAttributeRoutes();

        config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
    }
}
LISTO! Ya se están inyectando nuestras dependencias de las clases de lógica de negocio en nuestros Controllers.

En el siguiente tutorial veremos cómo logramos que se inyecten el resto de las dependencias en otras capas de nuestra app.

Tutorial II: Unity para el resto de nuestro sistema
Por ahora solo estamos inyectando nuestras clases de la lógica de negocio en nuestros controllers de Web Api. Lo que veremos aquí es cómo haremos para inyectar el resto de las dependencias. Para ello tenemos dos opciones:

Inyectar las dependencias del mismo modo en que lo hicimos anteriormente. En la clase WebApiConfig.cs registraríamos otras dependencias, por ejemplo nuestros repositorios que son usados por nuestras clases de la lógica de negocio.
Utilizar una capa que sea transversal a toda nuestra solución y que se encargue de crear las dependencias a partir de la lógica establecida para resolverlas.
Seguiremos la segunda opción.

1. Crearemos un nuevo proyecto Lupi.DependencyResolver
Será el encargado de resolver las dependencias del resto de nuestros objetos. Será del tipo biblioteca de clases.

2. Instalar Unity
A tal proyecto le instalamos Unity como paquete de Nuget sobre el proyecto Lupi.Web.Api

También podemos hacerlo a través de la Consola de Administrador de Paquetes:

Install-Package Unity
3. Agregar referencia a System.ComponentModel.Composition
Click derecho al proyecto -> Agregar -> Referencia -> Framework / Ensamblados -> System.ComponentModel.Composition;

4. Agregamos nuestras interfaces: IComponentResolver e IComponent
Agregaremos una interfaz llamada IComponent a nuestro proyecto DependencyResolver. Esta contendrá simplemente un método de inicialización llamado Setup. Esta interfaz ser implementada en nuestros proyectos, indicando la forma en que resolven las dependencias para cada proyecto. Por ejemplo tendremos un Resolver para BusinessLogic y otro para Repository. También modificaremos el ya existente de nuestra Web Api.

IComponent:

namespace Lupi.DependencyResolver
{
    public interface IComponent
    {
        void SetUp(IRegisterComponent registerComponent);
    }
}
Esta interfaz depende de IRegisterComponent. Esta será quien funcione como contrato para registrar nuestros componentes.

IRegisterComponent:

namespace Lupi.DependencyResolver
{
    public interface IRegisterComponent
    {
        void RegisterType<TFrom, TTo>(bool withInterception = false) where TTo : TFrom;
        void RegisterTypeWithControlledLifeTime<TFrom, TTo>(bool withInterception = false) where TTo : TFrom;
    }
}
5. Agregamos la clase que se encargará de registrar los tipos en el contenedor: ComponentResolver
Esta clase, llamémosla ComponentResolver, cargará tipos dinámicamente mediante Reflection. Aún no hemos visto este concepto, pero básicamente es una técnica para cargar ensamblados (y hacer cosas con ellos como instanciar objetos de sus clases), de forma dinámica, a través de llamadas a funciones. Esto evita tener que tener las dependencias hardcodeadas, aunque es más costoso porque se hace en tiempo de ejecución.

Esta clase, ComponentResolver, lo que hará es cargar componentes desde un cierto contenedor, leyendo ensamblados por Reflection en una cierta ruta (path) especificada.

ComponentResolver

using System;
using System.Collections.Generic;
using System.ComponentModel.Composition.Hosting;
using System.ComponentModel.Composition.Primitives;
using System.Linq;
using System.Reflection;
using System.Text;
using Microsoft.Practices.Unity;

namespace Lupi.DependencyResolver
{
    public class ComponentLoader
    {
        public static void LoadContainer(IUnityContainer container, string path, string pattern)
        {
            var dirCat = new DirectoryCatalog(path, pattern);
            var importDef = BuildImportDefinition();
            try
            {
                using (var aggregateCatalog = new AggregateCatalog())
                {
                    aggregateCatalog.Catalogs.Add(dirCat);

                    using (var componsitionContainer = new CompositionContainer(aggregateCatalog))
                    {
                        IEnumerable<Export> exports = componsitionContainer.GetExports(importDef);

                        IEnumerable<IComponent> modules = exports.Select(export => export.Value as IComponent).Where(m => m != null);

                        var registerComponent = new RegisterComponent(container);
                        foreach (IComponent module in modules)
                        {
                            module.SetUp(registerComponent);
                        }
                    }
                }
            }
            catch (ReflectionTypeLoadException typeLoadException)
            {
                var builder = new StringBuilder();
                foreach (Exception loaderException in typeLoadException.LoaderExceptions)
                {
                    builder.AppendFormat("{0}\n", loaderException.Message);
                }

                throw new TypeLoadException(builder.ToString(), typeLoadException);
            }
        }

        private static ImportDefinition BuildImportDefinition()
        {
            return new ImportDefinition(
                def => true, typeof(IComponent).FullName, ImportCardinality.ZeroOrMore, false, false);
        }
    }

    internal class RegisterComponent : IRegisterComponent
    {
        private readonly IUnityContainer _container;

        public RegisterComponent(IUnityContainer container)
        {
            this._container = container;
            //Register interception behaviour if any
        }

        public void RegisterType<TFrom, TTo>(bool withInterception = false) where TTo : TFrom
        {
            if (withInterception)
            {
                //register with interception
            }
            else
            {
                this._container.RegisterType<TFrom, TTo>();
            }
        }

        public void RegisterTypeWithControlledLifeTime<TFrom, TTo>(bool withInterception = false) where TTo : TFrom
        {
            this._container.RegisterType<TFrom, TTo>(new ContainerControlledLifetimeManager());
        }
    }
}
6. Compilar y agregar referencias desde el resto de los proyectos.
Recordemos que este proyecto es una capa transversal que se encarga de tomar control para inyectar las dependencias. En consecuencia, todos los proyectos deberán apuntar a Lupi.DependencyResolver. Agregamos la referencia desde Lupi.Repository, Lupi.BusinessLogic y Lupi.Web.Api a Lupi.Dependency Resolver.

7. Creamos un Resolver concreto para uno de los proyectos: Lupi.BusinessLogic
Crearemos una clase concreta que implemente la interfaz IComponent que dijimos que ibamos a implementar en cada uno de nuestros proyectos.

Le llamaremos DependencyResolver.

using Lupi.DependencyResolver;
using System.ComponentModel.Composition;
namespace Lupi.BusinessLogic
{
    [Export(typeof(IComponent))]
    public class DependencyResolver : IComponent
    {
        public void SetUp(IRegisterComponent registerComponent)
        {
            registerComponent.RegisterType<IBreedsBusinessLogic, BreedsBusinessLogic>();
            //Aca registraríamos otros tipos.
        }
    }
}
Hemos implementado el metodo SetUp y en el mismo método utilizamos nuestro registerComponent para registrar los tipos que precisamos para ese ensamblado.

Para el resto de los proyectos es igual, en el código de este repositorio se podrá apreciar lo mismo.

7. Cambiar el método Register para que dispare la lógica desde la Web.Api
using Lupi.DependencyResolver;
using Microsoft.Practices.Unity;
using System.Web.Http;

namespace Lupi.Web.Api
{
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {
            // Configuración y servicios de API web

            var container = new UnityContainer();

            ComponentLoader.LoadContainer(container, ".\\bin", "Lupi.*.dll");
            
            config.DependencyResolver = new UnityResolver(container);

            // Rutas de API web
            config.MapHttpAttributeRoutes();

            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );
        }
    }
}
Lo que hace la línea c# ComponentLoader.LoadContainer(container, ".\\bin", "Lupi.*.dll"); es simplemente cargar las dependencias registradas por cada proyecto en su DependencyResolver (dentro del metodo SetUp). Esto lo hace cargando por Reflection los ensamblados que hay dentro de la carpeta bin y que se llaman *Lupi.NOMBREDELENSAMBLADO.dll

8. Listo! Ya tenemos todo pronto.
Ahora siempre que queramos agregar una nueva dependencia deberemos registrarla dentro de la clase DependencyResolver, en cada proyecto.

Hemos logrado invertir el control y centralizar nuestras dependencias en un único lugar.
