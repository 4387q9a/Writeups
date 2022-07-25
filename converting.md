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