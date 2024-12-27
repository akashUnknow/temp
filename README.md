using System;
using System.Collections.Generic;
using System.IO;
using System.Text.RegularExpressions;
using System.Threading;

class Program
{
    static void Main()
    {
        // Get the current directory
        string currentDirectory = Directory.GetCurrentDirectory();

        // Move up two directories
        string parentDirectory = Directory.GetParent(Directory.GetParent(currentDirectory).FullName).FullName;

        // Define the processed folder path
        string processedFolder = Path.Combine(parentDirectory, "Processed");

        // Define the database folder path
        string databaseFolder = Path.Combine(parentDirectory, "Database");

        // Create the Processed and Database folders if they don't exist
        if (!Directory.Exists(processedFolder))
        {
            Directory.CreateDirectory(processedFolder);
            Console.WriteLine($"Processed folder created at: {processedFolder}");
        }

        if (!Directory.Exists(databaseFolder))
        {
            Directory.CreateDirectory(databaseFolder);
            Console.WriteLine($"Database folder created at: {databaseFolder}");
        }

        // Define a single output file name
        string outputFileName = "output.txt";
        string outputFilePath = Path.Combine(databaseFolder, outputFileName);

        // Loop to continuously check for .txt files
        while (true)
        {
            // Get all .txt files in the parent directory (two levels up)
            string[] txtFiles = Directory.GetFiles(parentDirectory, "*.txt");

            // If .txt files exist, process them
            if (txtFiles.Length > 0)
            {
                Console.WriteLine("Processing .txt files...");

                foreach (string file in txtFiles)
                {
                    try
                    {
                        // Read the file content
                        string fileContent = File.ReadAllText(file);

                        // Extract IMSI and ICCID
                        string imsi = ExtractIMSI(fileContent);
                        string iccid = ExtractICCID(fileContent);

                        // Extract quantity from the file content
                        int quantity = ExtractQuantity(fileContent);

                        // Check if duplicates exist in the database output file before writing
                        if (CheckForDuplicateIMSICCID(outputFilePath, imsi, iccid))
                        {
                            Console.WriteLine($"Error: Duplicate IMSI or ICCID found in the database: IMSI = {imsi}, ICCID = {iccid}");
                            continue; // Skip this file and move to the next one
                        }

                        // Append the IMSI and ICCID for the defined quantity to the single output file
                        using (StreamWriter writer = new StreamWriter(outputFilePath, append: true))
                        {
                            for (int i = 0; i < quantity; i++)
                            {
                                writer.WriteLine($"IMSI: {imsi}");
                                writer.WriteLine($"ICCID: {iccid}");
                                // Increment IMSI and ICCID for the next iteration
                                imsi = IncrementIMSI(imsi);
                                iccid = IncrementICCID(iccid);
                            }
                        }

                        Console.WriteLine($"IMSI and ICCID written to: {outputFilePath}");

                        // Move the original file to the Processed folder after successful processing
                        string destinationPath = Path.Combine(processedFolder, Path.GetFileName(file));
                        File.Move(file, destinationPath);
                        Console.WriteLine($"Moved: {file} to {destinationPath}");

                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error processing file {file}: {ex.Message}");
                    }
                }
            }
            else
            {
                Console.WriteLine("No .txt files found in the directory.");
            }

            // Wait for 5 seconds before checking again
            Thread.Sleep(5000); // Wait 5 seconds
        }
    }

    // Method to extract IMSI using a regular expression
    static string ExtractIMSI(string content)
    {
        // Example: Extract a 15-digit IMSI (you can adjust the pattern depending on your needs)
        Regex imsiRegex = new Regex(@"\b\d{15}\b");
        Match match = imsiRegex.Match(content);
        return match.Success ? match.Value : "IMSI not found";
    }

    // Method to extract ICCID using a regular expression
    static string ExtractICCID(string content)
    {
        // Example: Extract a 19-20 digit ICCID (you can adjust the pattern depending on your needs)
        Regex iccidRegex = new Regex(@"\b\d{19,20}\b");
        Match match = iccidRegex.Match(content);
        return match.Success ? match.Value : "ICCID not found";
    }

    // Method to extract quantity from the file content
    static int ExtractQuantity(string content)
    {
        // Look for a line like "Quantity: <number>"
        Regex quantityRegex = new Regex(@"Quantity:\s*(\d+)");
        Match match = quantityRegex.Match(content);
        if (match.Success && int.TryParse(match.Groups[1].Value, out int quantity))
        {
            return quantity;
        }

        Console.WriteLine("Quantity not found in the file. Defaulting to 1.");
        return 1; // Default to 1 if quantity is not found
    }

    // Method to check if the IMSI or ICCID already exists in the output file
    static bool CheckForDuplicateIMSICCID(string outputFilePath, string imsi, string iccid)
    {
        // Check if the output file exists
        if (File.Exists(outputFilePath))
        {
            try
            {
                // Read the content of the output file
                string content = File.ReadAllText(outputFilePath);

                // Check if IMSI or ICCID already exists in the output file
                if (content.Contains(imsi) || content.Contains(iccid))
                {
                    Console.WriteLine($"Duplicate found in output file: {outputFilePath}");
                    Console.WriteLine($"Duplicate IMSI: {imsi}, Duplicate ICCID: {iccid}");
                    return true; // Duplicate found
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error checking duplicates in output file {outputFilePath}: {ex.Message}");
            }
        }

        return false; // No duplicates found
    }

    // Method to increment IMSI by 1 (as a string)
    static string IncrementIMSI(string imsi)
    {
        if (long.TryParse(imsi, out long imsiNum))
        {
            imsiNum++;
            return imsiNum.ToString("D15"); // Ensure it stays 15 digits long
        }
        return imsi; // Return original if unable to parse
    }

    // Method to increment ICCID by 1 (as a string)
    static string IncrementICCID(string iccid)
    {
        if (long.TryParse(iccid, out long iccidNum))
        {
            iccidNum++;
            return iccidNum.ToString("D19"); // Ensure it stays 19 digits long
        }
        return iccid; // Return original if unable to parse
    }
}
