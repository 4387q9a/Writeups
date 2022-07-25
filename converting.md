# Converting 1.5.2 Player Data to 1.8 - A Journey of Pain, Hopelessness and Rage

This whole fiasco started a couple of days ago when ayunami2000 pushed a patch to EaglercraftBungee which fixed 1.8 ProtocolSupport servers.

Following this, and after performing a playtest with players on my server, I decided to switch our main servers over to 1.8.

I started off by switching KitPvP to 1.8. There were some small hitches in the beginning, but after recompiling the KitPvP plugin for 1.8 and making some minor adjustments to Essentials, and switching to a new permissions plugin, we were back in business.

After this, I decided to upgrade Factions. This is where the problems began. It started off by trying to start the server. Immediately, Spigot started to complain about the playerdata files. Unfortunately I don't have the logs on hand with me anymore, but at this point I was a little bit confused, and maybe raged a small amount.

## Converting 1.5.2 world data

At first, the process looked fairly simple. Generate some random UUID, rename the player data file, and that would be it. Right?

I thought so, but upon attempting that, I realised immediately that it would not work. After further research (including decompiling Spigot 1.8.8 jar files and Googling) I determined that these were not just randomly generated UUIDs - these were generated from the player's username. At this point, I was starting to get a bit mad, since it was alraedy half 3 in the morning.

Fast forward around half an hour later, I had written a basic Java program, that included the decompiled Spigot jar method for calculating the UUID. This also worked for those in offline mode, which was great.

The code at this point can be found below. Atrocious.

```java
import java.io.File;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.UUID;

public class Main {

    private static UUID bytesToUUID(byte[] bytes) {
        ByteBuffer buffer = ByteBuffer.wrap(bytes);
        long high = buffer.getLong();
        long low = buffer.getLong();
        return new UUID(high, low);
    }

    private static UUID nameUUIDFromBytes(byte[] name) {
        MessageDigest md;
        try {
            md = MessageDigest.getInstance("MD5");
        }
        catch (NoSuchAlgorithmException nsae) {
            throw new InternalError("MD5 not supported", nsae);
        }
        byte[] digest;
        byte[] md5Bytes = digest = md.digest(name);
        int n = 6;
        digest[n] &= 0xF;
        int n2 = 6;
        md5Bytes[n2] |= 0x30;
        int n3 = 8;
        md5Bytes[n3] &= 0x3F;
        int n4 = 8;
        md5Bytes[n4] |= (byte)128;
        return bytesToUUID(md5Bytes);
    }

    public static void main(String[] args) {
        if (args.length < 1) {
            System.out.println("Usage: Experiment.jar [dirName]");
            System.out.println("Outputs all converted files to playerdata/");
            return;
        }
        File directory = new File(args[0]);
        File[] directoryListing = directory.listFiles();
        if (directoryListing == null) {
            System.out.println("Directory name specified was not a directory.");
            return;
        }
        for (File file : directoryListing) {
            String[] splitName = file.getName().split("\\.");
            if (splitName.length < 1) continue; // Ignore because not a valid file
                                                // All player data files end in .dat
            String playerName = splitName[0];
            System.out.println(nameUUIDFromBytes(
                    ("OfflinePlayer:" + playerName).getBytes(StandardCharsets.UTF_8)) + " (" + playerName + ")");
        }
    }
}
```

This was overall, fairly simple to do. It took a while, there was a lot of awful code involved and caused me a great headache, however this was not the worst part of the migration. The worst part so far has been the Essentials conversion.

## Converting 1.5.2 Essentials userdata to 1.8 Essentials userdata

Looking at the way the two versions of the plugin stored the file, I believed the process would be very similar. However, after further inspection, I realised that Essentials 1.8 also included other things, such as UUIDs for worlds as opposed to just names, and also storing the account name.

This complicated things a little bit, as I was not entirely sure how I was going to approach it at first. My original plan was just to use some kind of YAML parser and update every file, adding a hardcoded world UUID to every single key that mentioned `world`. Ultimately, I decided against this, since I believed there was probably a smarter approach. One that didn't involve something so unnecessary.

I began writing code that would've behaved similarly to converting 1.5.2 world data, just to see what would happen. But as I was in the middle of writing this code, I also realised that Essentials for 1.5.2 stored all user data **in lower case**. This would not be that big of a deal, however, when you consider that the UUID generation algorithm is case sensitive, this becomes a big issue.

I decided to try it anyways, and lo and behold, it was like everybody's data got reset, since the only way (at that point) to access your data would be to play with your lowercase username. I ultimately decided this was sub-optimal and came up with other ways to approach this.

At this point, things started to get more and more specific to code that I had already written. My new plan of attack was going to be just simply getting a list of all player names in the old player data folder for the 1.5.2 world, and then using that (because it had the proper capitalisations) to find the correct Essentials userdata files and properly convert them over.

The code of which can be found below.

```java
// TODO: Add code (lol)
```

By using `newPlayerName`, I could keep the same capitalisation as the original username, allowing me to then use the algorithm to generate the proper UUID. Following this, I did this for all userdata files, and then uploaded the result to the Factions server and restarted.

Everything looked fine.. until I tried to run /baltop.

Immediately, console began flooding with error after error. You can see a snippet of the console errors below:
```
	
[03:46:43] [Server thread/INFO]: Cold issued server command: /baltop
[03:46:43] [Craft Scheduler Thread - 1/ERROR]: [Essentials] Error while reading user config: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
com.earth2me.essentials.libs.configurate.serialize.CoercionFailedException: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:52) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:23) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.ScalarSerializer.deserialize(ScalarSerializer.java:115) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.AbstractListChildSerializer.deserialize(AbstractListChildSerializer.java:58) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load0(ObjectMapperImpl.java:64) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load(ObjectMapperImpl.java:48) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperFactoryImpl.deserialize(ObjectMapperFactoryImpl.java:204) ~[?:?]
	at com.earth2me.essentials.libs.configurate.AbstractConfigurationNode.get(AbstractConfigurationNode.java:151) ~[?:?]
	at com.earth2me.essentials.libs.configurate.ConfigurationNode.get(ConfigurationNode.java:520) ~[?:?]
	at com.earth2me.essentials.UserData.reloadConfig(UserData.java:86) ~[?:?]
	at com.earth2me.essentials.UserData.<init>(UserData.java:59) ~[?:?]
	at com.earth2me.essentials.User.<init>(User.java:97) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:180) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:30) ~[?:?]
	at com.google.common.cache.LocalCache$LoadingValueReference.loadFuture(LocalCache.java:3524) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.loadSync(LocalCache.java:2317) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.lockedGetOrLoad(LocalCache.java:2280) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.get(LocalCache.java:2195) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.get(LocalCache.java:3934) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.getOrLoad(LocalCache.java:3938) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$LocalLoadingCache.get(LocalCache.java:4821) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.earth2me.essentials.UserMap.getUser(UserMap.java:129) ~[?:?]
	at com.earth2me.essentials.BalanceTopImpl.calculateBalanceTopMap(BalanceTopImpl.java:32) ~[?:?]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftTask.run(CraftTask.java:59) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftAsyncTask.run(CraftAsyncTask.java:53) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_312]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_312]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_312]
[03:46:43] [Craft Scheduler Thread - 1/ERROR]: [Essentials] Error while reading user config: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
com.earth2me.essentials.libs.configurate.serialize.CoercionFailedException: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:52) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:23) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.ScalarSerializer.deserialize(ScalarSerializer.java:115) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.AbstractListChildSerializer.deserialize(AbstractListChildSerializer.java:58) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load0(ObjectMapperImpl.java:64) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load(ObjectMapperImpl.java:48) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperFactoryImpl.deserialize(ObjectMapperFactoryImpl.java:204) ~[?:?]
	at com.earth2me.essentials.libs.configurate.AbstractConfigurationNode.get(AbstractConfigurationNode.java:151) ~[?:?]
	at com.earth2me.essentials.libs.configurate.ConfigurationNode.get(ConfigurationNode.java:520) ~[?:?]
	at com.earth2me.essentials.UserData.reloadConfig(UserData.java:86) ~[?:?]
	at com.earth2me.essentials.UserData.<init>(UserData.java:59) ~[?:?]
	at com.earth2me.essentials.User.<init>(User.java:97) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:180) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:30) ~[?:?]
	at com.google.common.cache.LocalCache$LoadingValueReference.loadFuture(LocalCache.java:3524) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.loadSync(LocalCache.java:2317) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.lockedGetOrLoad(LocalCache.java:2280) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.get(LocalCache.java:2195) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.get(LocalCache.java:3934) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.getOrLoad(LocalCache.java:3938) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$LocalLoadingCache.get(LocalCache.java:4821) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.earth2me.essentials.UserMap.getUser(UserMap.java:129) ~[?:?]
	at com.earth2me.essentials.BalanceTopImpl.calculateBalanceTopMap(BalanceTopImpl.java:32) ~[?:?]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftTask.run(CraftTask.java:59) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftAsyncTask.run(CraftAsyncTask.java:53) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_312]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_312]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_312]
[03:46:44] [Craft Scheduler Thread - 1/ERROR]: [Essentials] Error while reading user config: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
com.earth2me.essentials.libs.configurate.serialize.CoercionFailedException: [ignore, 0] of type java.util.List<java.util.UUID>: Failed to coerce input value of type class java.lang.String to UUID
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:52) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.UuidSerializer.deserialize(UuidSerializer.java:23) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.ScalarSerializer.deserialize(ScalarSerializer.java:115) ~[?:?]
	at com.earth2me.essentials.libs.configurate.serialize.AbstractListChildSerializer.deserialize(AbstractListChildSerializer.java:58) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load0(ObjectMapperImpl.java:64) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperImpl.load(ObjectMapperImpl.java:48) ~[?:?]
	at com.earth2me.essentials.libs.configurate.objectmapping.ObjectMapperFactoryImpl.deserialize(ObjectMapperFactoryImpl.java:204) ~[?:?]
	at com.earth2me.essentials.libs.configurate.AbstractConfigurationNode.get(AbstractConfigurationNode.java:151) ~[?:?]
	at com.earth2me.essentials.libs.configurate.ConfigurationNode.get(ConfigurationNode.java:520) ~[?:?]
	at com.earth2me.essentials.UserData.reloadConfig(UserData.java:86) ~[?:?]
	at com.earth2me.essentials.UserData.<init>(UserData.java:59) ~[?:?]
	at com.earth2me.essentials.User.<init>(User.java:97) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:180) ~[?:?]
	at com.earth2me.essentials.UserMap.load(UserMap.java:30) ~[?:?]
	at com.google.common.cache.LocalCache$LoadingValueReference.loadFuture(LocalCache.java:3524) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.loadSync(LocalCache.java:2317) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.lockedGetOrLoad(LocalCache.java:2280) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$Segment.get(LocalCache.java:2195) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.get(LocalCache.java:3934) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache.getOrLoad(LocalCache.java:3938) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.google.common.cache.LocalCache$LocalLoadingCache.get(LocalCache.java:4821) ~[BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at com.earth2me.essentials.UserMap.getUser(UserMap.java:129) ~[?:?]
	at com.earth2me.essentials.BalanceTopImpl.calculateBalanceTopMap(BalanceTopImpl.java:32) ~[?:?]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftTask.run(CraftTask.java:59) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at org.bukkit.craftbukkit.v1_8_R3.scheduler.CraftAsyncTask.run(CraftAsyncTask.java:53) [BurritoSpigot.jar:git-BurritoSpigot-"cb667b1"]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_312]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_312]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_312]
```

After these errors were spat out, /baltop was very interesting because it just contained a bunch of "null" where the username would be. This was very confusing.

Looking at the log, you can see that it's clearly not very happy with something to do with Strings and UUIDs. I was confused, because looking at a git diff between the 1.5.2 and 1.8 userdata files, I couldn't spot anything to do with UUIDs in there, except I did spot one field. `last-account-name`.

I believed this was responsible for the majority of the issues, and so, after a while, I added some code to my program that would append the last known account name to the end of the YAML file, in hopes that it would fix the error. After all, I assumed that the error was talking about not being able to convert some string to a UUID. It made sense in my head.

After re-running the program, uploading the new userdatas, and restarting the server, it was still producing the exact same error logs, however the usernames in /baltop were no longer just null, so at least there was some progress there.

Though, at this point, I was basically banging my head against my keyboard. It was around 4 AM at this point, and I just wanted to go to bed. But, alas, I carried on, plotting on how to tackle this in the best way possible. This stumped me for a couple of hours - I tried a couple of different methods, including using a library called SnakeYAML, and then just removing it manually. Ultimately, I decided against this because I really did not want to sift through possibly hundreds of user data files removing people's list of ignored players.

So, instead, as a last ditch effort, I turned to writing a Bukkit plugin. The concept of the plugin was very simple: iterate through every file in the Essentials userdata, use pre-existing YAML file parsers that Bukkit provides to remove the ignored field, and then saving the file.

The code for this plugin can be found below. 
```java
    public void onEnable() {
        File dir = new File("/home/container/plugins/Essentials/userdata");
        if (!dir.exists()) return;
        File[] files = dir.listFiles();
        if (files == null) return;
        for (File file : files) {
            FileConfiguration config = YamlConfiguration.loadConfiguration(file);
            if (!config.contains("ignore")) continue;
            config.set("ignore", null);
            try {
                config.save(file);
                getLogger().info("updated " + file.getName() + " and removed ignore list");
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
```

And, finally, after restarting the server the next time and running /baltop, there were no more errors. Additionally, it restored the position of some players on the baltop. Overall, a success. I was ecstatic. It was at this point I decided to turn this into a writeup as well, rather than just something I was documenting on the Eaglercraft Discord and to friends in group chats.

Overall, this process so far has been tiring, exhausting and has caused me multiple headaches. It should be noted that this type of thing does not happen often - this is a very specific case where I cannot just reset my server back to day one, where people have already started building bases and exploring the map. Resetting is an absolute last resort.

Now, the next thing to migrate is Factions. Only time will tell how that will go. (although, it does appear that there is actual documentation for this, which is good news!)

## Migrating Factions data from 1.5.2 to 1.8

To be continued.