## Project Structure 

* **.csproj:** The configuration file for a single program or library. Gatehouse.Server will be its own project, holding all the settings, external NuGet packages, and build instructions needed to compile and run the server.
* **.slnx:** A container file that groups related projects together so they can be opened, managed, and built as one unified system.
* **global.json:** Locks in the specific .NET SDK version for the repository so anyone cloning the project builds it with the exact same tooling.
* **Directory.Build.props:** This holds the settings for every project 
* **Directory.Packages.props:** Enables (CPM) so every NuGet package version is defined in this single file
