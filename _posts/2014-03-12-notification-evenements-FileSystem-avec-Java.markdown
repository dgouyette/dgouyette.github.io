---
layout: post
title:  "Notification d'évènements dans le FileSystem avec Java > 5"
date:   2014-03-12 09:00:00
categories: jekyll update
---
Pourquoi surveiller les Events du FileSystem ?

Je travaille actuellement avec spring-integration.
Celui-ci est chargé de renommer puis déplacer des fichiers arrivant dans un répertoire d'entrée dans le répertoire de sortie (en réalité, c'est un poil plus compliqué).


**In : toto.txt ==> Out : 2014-03-12—07-00-59-toto.txt**

Si je souhaite écrire un test d’intégration robuste, qui ne va pas
échouer parce que le test prend trop de temps à s’exécuter, cela va être
difficile.

Je ne peux plus faire ainsi :

{% highlight java %}
//aFile = toto.txt
FileUtils.waitFor(aFile, 5)
{% endhighlight %}



Cette API (commons-io), si vous ne la connaissez pas, fais une boucle
jusqu’à ce le délai imparti (en secondes) soit écoulé, ou alors que le
fichier apparaisse.
Elle retourne ensuite true/false selon la présence du fichier.

La même chose n’existe pas lorsque l’on ne connait pas à l’avance le nom
du fichier. Elle n’existe pas non plus dans le cas ou l’on veut
s’assurer de la disparition du fichier.

Par contre depuis java7 il est possible d’être notifié des évènements du
FileSystem cf
[http://docs.oracle.com/javase/tutorial/essential/io/notification.html](http://docs.oracle.com/javase/tutorial/essential/io/notification.html)

Je n’ai pas réussi à faire fonctionner simplement le code de l’URL ci
dessus à la fois sur mon OS (MacOS) et sur l’usine d’intégration de
l’entreprise (Debian)
J’avais eu le même problème quelques jours auparavant, quand j’ai tenté
d’utiliser NIO2 pour ajouter des métadatas (type User) à un fichier, ok
sous ubuntu, ko sous mac

J’ai lancé un appel à l’aide sur twitter, espèrant trouver la *“silver
bullet”*, une méthode inconnue de Guava, ou de commons-io utilisant ce
système de notifications sur le FS.

La solution
-----------

Après une bonne nuit de repos sans toucher à mon clavier (un peu de
tablette quand même, j’avoue). J’ai finalement trouvé la librairie
commons-vfs2 d’Apache, et celle ci offre la même chose, mais via un
système de fichier virtualisé, qui permet de s’abstraire des
contingences propres aux différents système de fichiers.

Cette librairie fournit un ensemble d’API permettant d’accèder à
différents système de fichiers
(http://commons.apache.org/proper/commons-vfs/filesystems.html) de
manière uniforme :

*  Zip,
*  HTTP,
*  Disque local,
* Gestion des évènements

**Elle fonctionne à partir de java 5**

Le Test d’intégration :
-----------------------

{% highlight java %}
`Test
public void should_observe_2_create_file_event() throws FileSystemException, InterruptedException {
        //given
        final File tempDir = Files.createTempDir();
        createTwoFileInThread(tempDir);
        //when
        List<FileChangeEvent> events = watch(tempDir.getAbsolutePath(), 2, 10);
        Collection<String> fileNames = transform(events, fileEventToString);
        //then
        assertThat(fileNames).hasSize(2);
        assertThat(fileNames).contains(EXPECTED_NAME_FILE1, EXPECTED_NAME_FILE2);
    }
{% endhighlight %}


Le code du watcher
-----------------------


{% highlight java %}
public class IOWatcher {
    static List<FileChangeEvent> watch(String pathToFile, int expectedEvents, int seconds) throws FileSystemException, InterruptedException {
        FileSystemManager fileSystemManager = VFS.getManager();
        FileObject dirToWatch = fileSystemManager.resolveFile(pathToFile);
        SimpleFileMonitoring monitoring = new SimpleFileMonitoring();
        DefaultFileMonitor fileMonitor = new DefaultFileMonitor(monitoring);
        fileMonitor.setRecursive(true);
        fileMonitor.addFile(dirToWatch);
        fileMonitor.start();
        DateTime expiration = new DateTime().plusSeconds(seconds);
        //jusqu'à expiration du timeout (maintenant + n seconds)
        while (new DateTime().isBefore(expiration)) {
            //ou juqu'à ce que le nombre d'évènements attendus
            if (monitoring.getIoEvents().size() >= expectedEvents) {
                break;
            }
            //boucle toutes les 100ms
            Thread.sleep(100);
        }
        fileMonitor.stop();
        return monitoring.getIoEvents();
    }
}
{% endhighlight %}



Le code du monitor de FS
-----------------------


{% highlight java %}
class SimpleFileMonitoring implements FileListener {
    //evenenements du FileSystem
    private final List<FileChangeEvent> ioEvents = new ArrayList<FileChangeEvent>();
    `Override
 public void fileCreated(FileChangeEvent fileChangeEvent) throws
Exception {
 ioEvents.add(fileChangeEvent);\
 }
 `Override
    public void fileDeleted(FileChangeEvent fileChangeEvent) throws Exception {
        ioEvents.add(fileChangeEvent);
    }
    `Override
 public void fileChanged(FileChangeEvent fileChangeEvent) throws
Exception {
 ioEvents.add(fileChangeEvent);
 }
 public List<FileChangeEvent> getIoEvents() {
 return ioEvents;
 }
}
{% endhighlight %}


Voilà pour ce premier article de l’année, j’espère qu’il pourra vous
être utile.
